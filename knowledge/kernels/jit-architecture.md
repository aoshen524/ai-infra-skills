<!-- source: FlashInfer -->

# JIT Compilation Architecture for GPU Kernels

## Why JIT

GPU kernel libraries face a combinatorial explosion of configurations:
- Head dimensions: 64, 96, 128, 192, 256, ...
- Data types: FP16, BF16, FP8 (E4M3, E5M2), ...
- Layouts: HND, NHD, packed, ...
- Features: causal mask, sliding window, paged KV, ...

AOT (ahead-of-time) compilation of all combinations leads to:
- Enormous binary sizes (hundreds of MB to GB)
- Long build times (hours for full matrix)
- Still can't cover user-defined combinations

JIT compiles only the specific kernel variants actually needed at runtime, trading first-call latency for minimal binary size and maximal flexibility.

## Core Pattern

The typical JIT architecture for GPU kernels:

```
Python API call (head_dim=128, dtype=bf16, layout=NHD)
    |
    v
Template Selection (Python)
    - Map runtime params to C++ template parameters
    - Generate kernel source string or select template file
    |
    v
Cache Lookup
    - Hash(arch, dtype, head_dim, layout, feature_flags) -> cache key
    - If .so/.cubin exists in cache dir -> dlopen and return
    |
    v
Compilation (on cache miss)
    - Write generated .cu source to temp file
    - Invoke nvcc/nvrtc with appropriate flags (-arch=sm_90a, -O2, etc.)
    - Compile to .so (nvcc) or .cubin (nvrtc)
    - Move to cache directory
    |
    v
Load and Execute
    - dlopen the .so / load .cubin via CUDA driver API
    - Extract kernel function pointer
    - Launch with computed grid/block dims
```

## Cache Management

### Hash-Based Cache Keys

The cache key must include everything that affects the compiled binary:

```python
cache_key = hashlib.sha256(
    f"{cuda_arch}|{dtype}|{head_dim}|{layout}|{features}|{source_hash}"
).hexdigest()
```

Critical components:
- **CUDA architecture** (sm_80, sm_90a): different ISA, different optimizations
- **All template parameters**: dtype, head_dim, layout, etc.
- **Source code hash**: invalidate cache when kernel source changes
- **Compiler flags**: optimization level, debug flags

### Persistent Cache Directory

```python
# Typical cache location
cache_dir = os.environ.get("KERNEL_CACHE_DIR",
                           os.path.expanduser("~/.cache/kernel_lib/"))

# Structure
# ~/.cache/kernel_lib/
#   a1b2c3d4.so      # cached compiled kernel
#   a1b2c3d4.cu      # (optional) preserved source
#   e5f6g7h8.so
#   ...
```

Cache should survive process restarts. First call in a new environment pays compilation cost; subsequent calls are fast dlopen.

## Template Instantiation

### Python-Side Config Enumeration

```python
# Define the space of possible configurations
@dataclass
class KernelConfig:
    head_dim: int
    dtype: str
    layout: str
    use_causal: bool

# At runtime, instantiate only what's needed
config = KernelConfig(head_dim=128, dtype="bf16", layout="NHD", use_causal=True)
kernel = jit_compile(config)
```

### C++ Template Generation

The Python layer generates C++ source with concrete template parameters:

```python
source = f"""
#include "attention_kernel.cuh"

// Explicit instantiation for this specific config
template void attention_kernel<
    /*HeadDim=*/{config.head_dim},
    /*DType=*/{dtype_map[config.dtype]},
    /*Layout=*/{layout_map[config.layout]},
    /*Causal=*/{'true' if config.use_causal else 'false'}
>(const KernelParams& params);
"""
```

This approach keeps the C++ kernel templated and generic while letting Python control which instantiations exist.

## Debugging JIT Kernels

### Keep Generated Source

Most JIT systems support preserving the generated source for inspection:

```bash
# FlashInfer
export FLASHINFER_JIT_DIR=/tmp/flashinfer_jit    # persistent cache dir
export FLASHINFER_VERBOSE=1                        # print compilation commands

# General pattern
export KERNEL_CACHE_DIR=/tmp/kernel_cache
export KERNEL_KEEP_SOURCE=1
```

### Inspect Cache Directory

```bash
# See what's been compiled
ls -la ~/.cache/kernel_lib/

# Find the source for a specific config
grep -r "head_dim=128" ~/.cache/kernel_lib/*.cu
```

### Force Recompilation

```bash
# Clear cache to force recompile (useful after source changes)
rm -rf ~/.cache/kernel_lib/

# Or set env var
export KERNEL_FORCE_RECOMPILE=1
```

### Common JIT Issues

1. **Permission errors**: Cache directory not writable in container environments. Fix: set `CACHE_DIR` to a writable path.
2. **nvcc not found**: CUDA toolkit not in PATH. Fix: set `CUDA_HOME` or add to PATH.
3. **Architecture mismatch**: Cached binary compiled for sm_80, running on sm_90. Fix: include arch in cache key (most systems do this already).
4. **Stale cache**: Source code changed but hash doesn't include source content. Fix: include source hash in cache key or clear cache.
5. **Concurrent compilation**: Multiple processes compiling the same kernel simultaneously. Fix: use file locks or atomic rename for cache writes.

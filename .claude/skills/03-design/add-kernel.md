# Kernel Development Patterns

Patterns for adding new CUDA kernels to a kernel library or inference framework, covering both JIT and AOT compilation paths.

<!-- source: FlashInfer (JIT/AOT CUDA kernels), SGLang (JIT kernel module), Liger-Kernel (benchmark best practices) -->

## Overview

A complete kernel addition involves these layers:

```
CUDA Kernel (.cuh)  -->  Launcher (.cu)  -->  FFI Binding  -->  JIT Generator  -->  Python API  -->  Tests  -->  Benchmark
```

Each layer has a clear responsibility. The kernel is framework-agnostic; the launcher handles tensor validation and dtype dispatch; the binding exports to the host language; the Python API provides the user-facing interface.

### Recommended Development Loop

For optimization-heavy kernel work, prefer an autoresearch-style loop:

1. freeze a trusted correctness and benchmark harness
1. constrain normal edits to the kernel implementation surface
1. make one meaningful optimization change per round
1. run correctness first, then performance
1. write the result to a persistent perf log before the next round

Why this matters:

- kernel work is measurable enough for iterative search to work well
- unconstrained edits to the benchmark harness create fake speedups
- small rounds are much easier to attribute than large rewrites

See `knowledge/kernels/kernel-autoresearch-loop.md` for the repo-level pattern.

---

## Step 1: Define the CUDA Kernel

Create a header file (`.cuh`) in the kernel include directory.

**Principles:**
- Framework-agnostic -- no PyTorch/TVM headers, only raw CUDA types and pointers.
- Template-based for dtype flexibility (`half`, `__nv_bfloat16`, `float`).
- Include only what is needed (`cuda_runtime.h`, `cuda_fp16.h`, `cuda_bf16.h`).

```cpp
template <typename T>
__global__ void MyKernel(const T* input, T* output, T factor, int n) {
  int idx = blockIdx.x * blockDim.x + threadIdx.x;
  if (idx < n) {
    output[idx] = input[idx] * factor;
  }
}

template <typename T>
cudaError_t MyKernelLauncher(const T* input, T* output, T factor, int n,
                             cudaStream_t stream = nullptr) {
  const int threads = 256;
  const int blocks = (n + threads - 1) / threads;
  MyKernel<T><<<blocks, threads, 0, stream>>>(input, output, factor, n);
  return cudaGetLastError();
}
```

<!-- source: FlashInfer -->

---

## Step 2: Create the Launcher

The launcher sits between the raw kernel and the FFI layer. It:

- Validates input tensors (device, dtype, contiguity, shape).
- Dispatches on dtype using macros or template specialization.
- Extracts the CUDA stream from the framework's tensor/device abstraction.
- Converts high-level tensor views to raw pointers for the kernel.
- Checks kernel launch status and provides descriptive error messages.

**Tensor validation patterns:**

Use the framework's validation utilities rather than manual checks:

```cpp
// TensorMatcher pattern (SGLang JIT style)
SymbolicSize N = {"num_elements"};
SymbolicDevice device;
device.set_options<kDLCUDA>();
TensorMatcher({N})
    .with_dtype<fp16_t>()
    .with_device<kDLCUDA>(device)
    .verify(dst)
    .verify(src);  // same shape/dtype/device as dst
```

```cpp
// Macro pattern (FlashInfer style)
CHECK_INPUT(input);
CHECK_INPUT(output);
DISPATCH_DLPACK_DTYPE_TO_CTYPE_FP32_FP16(input.dtype(), DType, [&] {
  // launch kernel with DType
});
```

<!-- source: FlashInfer, SGLang -->

**Use vectorized memory access** for bandwidth-bound kernels:

```cpp
// 128-bit vectorized loads/stores
constexpr int kVecN = 16 / sizeof(T);  // 8 for fp16, 4 for fp32
AlignedVector<T, kVecN> v;
v.load(src, vec_idx);
// process...
v.store(dst, vec_idx);
```

<!-- source: SGLang -->

---

## Step 3: Create FFI Binding

Export the launcher function so it is callable from Python:

```cpp
// Forward declaration
void my_launcher(TensorView input, TensorView output, float factor);

// Export
TVM_FFI_DLL_EXPORT_TYPED_FUNC(run, my_launcher);
```

The export name (`run`) becomes the Python-callable method name on the compiled module.

<!-- source: FlashInfer -->

---

## Step 4: JIT Generator

The JIT generator manages compilation and caching of kernel variants.

**Key design points:**
- Generate a unique URI for each kernel configuration (dtype, specialization flags).
- Copy or template-generate source files into a build directory.
- Never write to package directories -- use a dedicated generation/cache directory.
- Return a build spec that the framework's JIT infrastructure can compile and load.

```python
# Simple case: no Jinja templating needed
def gen_my_module(dtype_in, dtype_out):
    uri = f"my_kernel_dtype_in_{dtype_in}_dtype_out_{dtype_out}"
    gen_directory = GEN_SRC_DIR / uri
    os.makedirs(gen_directory, exist_ok=True)
    # Copy source files
    for fname in ["my_kernel.cu", "my_kernel_binding.cu"]:
        shutil.copy(CSRC_DIR / fname, gen_directory / fname)
    return gen_jit_spec(name=uri, sources=[...], extra_cuda_cflags=[])
```

**For architecture-specific kernels**, specify supported SM versions:

| Supported Versions | Use Case |
|-------------------|----------|
| All / `None` | Universal kernels |
| SM90+ | Hopper-specific features |
| SM100+ | Blackwell-specific features |

<!-- source: FlashInfer -->

---

## Step 5: Python API

The Python API is the user-facing interface. It should be thin and clean.

**Principles:**
- Cache compiled modules (use `functools.cache` or framework-specific caching like `cache_once`).
- **Destination passing style**: accept optional `out` tensor parameter to allow pre-allocated buffers.
- Validate inputs in Python before calling the compiled kernel.
- Use decorator-based validation when the framework provides it (e.g., `@backend_requirement`, `@supported_compute_capability`).
- Add a clean docstring with parameters, return type, and usage example.

```python
@functools.cache
def _get_module(dtype_in, dtype_out):
    return gen_my_module(dtype_in, dtype_out).build_and_load()

def my_op(input: torch.Tensor, factor: float,
          out: torch.Tensor | None = None) -> torch.Tensor:
    if out is None:
        out = torch.empty_like(input)
    module = _get_module(str(input.dtype), str(out.dtype))
    module.run(input, out, float(factor))
    return out
```

<!-- source: FlashInfer, SGLang -->

---

## Step 6: Testing

### Required Tests

| Test Type | Purpose |
|-----------|---------|
| Correctness | Compare kernel output against PyTorch reference across dtypes and sizes |
| Pre-allocated output | Verify `out` parameter works and returns the same tensor |
| Error cases | CPU tensor, unsupported dtype, shape mismatch |
| Edge cases | Size 1, non-aligned sizes (tail elements), large tensors |

### Tolerance Guidelines

| dtype | rtol | atol |
|-------|------|------|
| float32 | 1e-5 | 1e-6 |
| float16 / bfloat16 | 1e-2 to 1e-3 | 1e-2 to 1e-3 |

### Correctness Structure

Recommended pattern:

- test forward parity against one canonical baseline
- test backward parity when the kernel participates in training
- reuse the same baseline later in benchmarks instead of writing a second one
- keep tolerance explicit per dtype and document any widened tolerance

### Architecture Gating

Skip tests on unsupported hardware:

```python
if not is_sm90a_supported(torch.device("cuda")):
    pytest.skip("Requires SM90a (Hopper) or later")
```

### CI Registration

If the project uses a test suite runner, register tests at module level with literal values for AST-based discovery:

```python
register_cuda_ci(est_time=30, suite="kernel-unit-1-gpu")
```

<!-- source: FlashInfer, SGLang -->

---

## Step 7: AOT Registration

Register common dtype/config combinations for ahead-of-time compilation so users with pre-built packages skip JIT:

```python
def gen_my_modules():
    for dtype in ["float16", "bfloat16", "float32"]:
        yield gen_my_module(dtype, dtype)
```

<!-- source: FlashInfer -->

---

## Step 8: Benchmarking

All new kernels should have benchmarks.

- Compare against PyTorch reference (and other implementations if relevant).
- Test across multiple sizes and dtypes.
- Use the framework's benchmarking utilities for consistent timing (CUDA events, CUPTI, or CUDA graphs).
- Report median and standard deviation.

### Recommended Benchmark Pattern

Use Liger-Kernel-style benchmarking as the default pattern:

- import the baseline implementation from tests
- build one shared setup path for speed and memory benchmarks
- keep setup and module construction outside the timed region
- benchmark `forward`, `backward`, and `full` separately when applicable
- run speed and peak-memory measurements in separate passes
- save exact repro metadata with provider, shape, dtype, and mode

For aggressive optimization loops, also adopt these guardrails:

- keep the evaluation harness effectively read-only once trusted
- log every optimization round with change, rationale, and measured outcome
- escalate to profiler, PTX, or SASS analysis after repeated no-gain rounds instead of guessing
- resume from `optimization_rules` and `perf_log` artifacts rather than relying on huge chat context

For training kernels, a benchmark is only complete when it covers:

- forward parity
- backward parity
- speed comparison
- memory comparison

<!-- source: FlashInfer, SGLang -->

---

## Summary of Files

A complete kernel addition typically creates/modifies:

```
include/or/csrc/  my_kernel.cuh          # CUDA kernel definition
csrc/             my_kernel.cu           # Launcher with tensor validation
csrc/             my_kernel_binding.cu   # FFI binding
jit/              my_kernel.py           # JIT generator
python_api/       my_kernel.py           # Python API
__init__.py                              # Export (modified)
aot.py                                   # AOT registration (modified)
tests/            test_my_kernel.py      # Unit tests
benchmarks/       bench_my_kernel.py     # Benchmark
```

---

## Common Pitfalls

- **Never write to package directories** from JIT -- use a dedicated generation/cache path.
<!-- source: FlashInfer -->
- **Handle scalar tail elements** when using vectorized loads -- tensors may not be aligned to the vector width.
<!-- source: SGLang -->
- **Use framework type aliases** (`fp16_t`, `bf16_t`, `fp32_t`) instead of raw CUDA types for consistency.
<!-- source: SGLang -->
- **Keep Python wrappers thin** but still validate basic preconditions (is_cuda, supported dtype, shape match) -- the JIT/FFI path may not reject invalid inputs safely.
<!-- source: SGLang -->
- **Use `cache_once` or `functools.cache`** for module compilation -- do not recompile on every call.
<!-- source: FlashInfer, SGLang -->

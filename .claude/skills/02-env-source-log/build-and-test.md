# Build, Install & Test Patterns

Synthesized patterns for building, installing, and testing GPU-accelerated AI infrastructure projects.

---

## Dependency Management Tools

### uv (Lockfile-Based)
<!-- source: Megatron-LM -->

`uv` provides deterministic, locked dependency resolution. Dependencies are declared in `pyproject.toml` with dependency groups for different use cases.

```bash
# Install full dev + test environment (inside container)
uv sync --locked --group dev --group test

# Install only linting tools
uv sync --locked --only-group linting

# Update lockfile after changing pyproject.toml
uv lock
```

**Dependency group pattern:**

| Group | Purpose |
|-------|---------|
| `dev` | Full dev environment (native extensions, GPU libs) |
| `test` | pytest, coverage, test helpers |
| `linting` | ruff, black, isort, pylint |
| `build` | Cython, pybind11, native build tools |
| `training` | Runtime training extras |

**Key rules:**
- Always run `uv sync --locked` inside the container, never on the bare host
- Never use bare `pip install` alongside uv -- use `uv add` and `uv sync`
- Commit `uv.lock` to version control for reproducibility
- Mount a host cache dir via `-v $HOME/.cache/uv:/root/.cache/uv` to avoid container disk pressure

### pip (requirements.txt)
<!-- source: torchtitan -->

For simpler projects, `pip` with requirements files is sufficient:

```bash
pip install -r requirements.txt -r requirements-dev.txt
```

**Key rules:**
- Pin versions in requirements files for CI reproducibility
- Use separate `requirements-dev.txt` for dev/test dependencies
- Prefer `pip install -e .` for editable installs of the project itself

---

## Editable Installs

### Standard Editable Install
<!-- source: torchtitan -->

```bash
pip install -e .
```

Changes to Python source are picked up immediately without reinstall.

### JIT-Based Editable Install (No Rebuild Needed for Kernel Changes)
<!-- source: FlashInfer -->

For projects using JIT (Just-In-Time) compilation, editable install means even CUDA kernel changes are picked up automatically:

```bash
# Initial install -- use --no-build-isolation to prevent pulling incompatible CUDA/PyTorch
pip install --no-build-isolation -e . -v
```

**How JIT compilation works (3-layer architecture):**

| Layer | Role | Example |
|-------|------|---------|
| **1. JitSpec** | Compilation metadata (sources, flags, unique hash) | `JitSpec(name=uri_hash, sources=[...], extra_cuda_cflags=[...])` |
| **2. Code Generation** | Render Jinja templates or copy source files to a writable gen directory | `gen_some_module(dtype_in, dtype_out, ...)` |
| **3. Compilation & Loading** | Generate `build.ninja`, compile with nvcc, load `.so` | `jit_spec.build_and_load()` |

**Cache behavior:**
- **First call**: Generates specialized CUDA code, compiles with ninja, caches `.so` file
- **Subsequent calls**: Uses cached compiled module
- **After kernel changes**: Automatically detects source changes (SHA256 hash) and recompiles

**Cache management:**

```bash
# Clear JIT cache
rm -rf ~/.cache/flashinfer/

# Override cache location
export FLASHINFER_WORKSPACE_BASE="/scratch"

# Control parallel compilation
export FLASHINFER_NVCC_THREADS=4

# Set target GPU architectures
export FLASHINFER_CUDA_ARCH_LIST="8.0 9.0a"
```

**JIT directory rules (critical):**

| Directory | Writable | Use for |
|-----------|----------|---------|
| Generated source dir | Yes | Jinja output, copied `.cu` files |
| JIT/cache dir | Yes | Compiled `.so` outputs |
| Source template dir | **No** | Read-only source templates (may be installed read-only) |

**Important**: Never write to package directories -- they may be read-only after installation.

### Submodule Initialization
<!-- source: FlashInfer -->

Projects with native dependencies often use git submodules (e.g., CUTLASS, spdlog):

```bash
# Clone with submodules
git clone --recursive <repo-url>

# Or initialize after clone
git submodule update --init --recursive
```

---

## Linting & Formatting

### pre-commit (Recommended)
<!-- source: torchtitan, FlashInfer -->

```bash
# Install hooks (run once)
pre-commit install

# Run all hooks manually
pre-commit run --all-files
# or
pre-commit run -a
```

**Rule**: Always run `pre-commit run --all-files` before opening a PR. CI linting failures waste everyone's time.

### Script-Based Linting
<!-- source: Megatron-LM -->

Some projects use custom linting scripts that wrap multiple tools:

```bash
# Check mode (no changes applied)
BASE_REF=main CHECK_ONLY=true bash tools/autoformat.sh

# Fix mode (apply changes)
BASE_REF=main CHECK_ONLY=false bash tools/autoformat.sh
```

Common tools invoked: `black`, `isort`, `ruff`, `pylint`, `mypy`.

---

## Test Execution

### pytest Patterns

```bash
# Run all tests
pytest tests/

# Run specific test file
pytest tests/path/test_file.py

# Run specific test function
pytest tests/path/test_file.py::test_function

# Stop at first failure
pytest -x tests/

# Verbose with stdout
pytest -xvs tests/path/test_file.py
```

### GPU Test Considerations
<!-- source: torchtitan, Megatron-LM -->

**Before running GPU tests**, check GPU availability:

```bash
nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv,noheader
# Empty output = GPUs free; non-empty = wait or ask
```

**Multi-GPU tests** use `torch.distributed.run`:
- Only ranks 0 and 3 are typically tee'd to stdout
- Other ranks write to per-rank log files
- Full per-rank logs are uploaded as CI artifacts

**Multi-GPU tests with MPI:**

```bash
mpirun -np 4 pytest tests/comm/test_allreduce_unified_api.py
```

### Skip Tests by GPU Architecture
<!-- source: FlashInfer -->

Use architecture check functions to skip tests on unsupported GPUs:

```python
def test_hopper_feature():
    if not is_sm90a_supported(torch.device("cuda")):
        pytest.skip("Requires SM90a (Hopper)")
    # Test code...
```

| Feature | Min SM | Typical Check |
|---------|--------|---------------|
| FlashAttention-3 / Hopper kernels | SM90a | `is_sm90a_supported()` |
| Blackwell kernels | SM100a | `is_sm100a_supported()` |
| FP8 GEMM | SM89+ | `get_compute_capability()[0] >= 9` |

### Validating Numerics
<!-- source: torchtitan -->

Non-computation changes (refactoring, activation checkpointing) must produce **identical loss** with deterministic seeds:

```bash
# Use debug flags for bit-wise reproducibility
--debug.seed=42 --debug.deterministic
```

Compare loss and grad_norm from TensorBoard, not just the 5-digit stdout truncation.

---

## Common Pitfalls

| Problem | Cause | Fix |
|---------|-------|-----|
| `uv sync --locked` fails | Stale `uv.lock` or dependency conflict | Run `uv lock` inside container, commit updated lock |
| `ModuleNotFoundError` after pip install | Installed outside the managed venv | Use `uv add` / `uv sync` or proper venv activation |
| JIT recompilation every run | Cache cleared or architecture mismatch | Check `FLASHINFER_CUDA_ARCH_LIST`, verify cache dir |
| Port collision on multi-GPU | torchrun binding conflicts | Use container entry point or explicit port assignment |
| Pre-commit fails | Code style violations | Run autoformat script or `pre-commit run -a` and commit fixes |
| `No space left on device` during builds | Cache fills container disk | Mount host cache dir into container |
| `--no-build-isolation` forgotten | pip pulls incompatible PyTorch/CUDA from PyPI | Always use `--no-build-isolation` for CUDA projects |

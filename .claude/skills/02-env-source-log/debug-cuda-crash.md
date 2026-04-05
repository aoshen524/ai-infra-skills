# CUDA Crash Debugging

<!-- source: FlashInfer, SGLang -->

Comprehensive guide for debugging CUDA crashes using API logging, compute-sanitizer, cuda-gdb, and kernel printf.

---

## 1. API Logging for CUDA Crash Debugging

**Problem**: CUDA errors crash the program before normal debugging output is flushed.

**Solution**: Many AI frameworks provide API-level logging decorators that capture inputs BEFORE kernel execution, so you can see what caused the crash even after the program aborts.

### Log Level Hierarchy

| Level | Content |
|-------|---------|
| 0 | No logging (default, zero overhead) |
| 1 | Function names only |
| 3 | Input/output metadata (shape, dtype, device, contiguity) |
| 5 | + Tensor statistics (min, max, mean, nan_count, inf_count) |
| 10 | + Crash-safe tensor dumps to disk |

### Log Destination

| Destination | When to Use |
|-------------|-------------|
| `stdout` | Quick interactive debugging |
| `stderr` | Separate from program output |
| `<file_path>` | **Crashes** — console output is lost when process aborts |
| `log_%i.txt` | Multi-process — `%i` expands to process ID |

### Example: FlashInfer

```bash
export FLASHINFER_LOGLEVEL=3
export FLASHINFER_LOGDEST=debug.log
python my_script.py
```

### Example: SGLang

```bash
export SGLANG_KERNEL_API_LOGLEVEL=3
export SGLANG_KERNEL_API_LOGDEST=debug.log
python my_script.py
```

### Level 10: Crash-Safe Tensor Dumps

At level 10, inputs are saved to disk BEFORE execution. If the kernel crashes, the dump directory still contains `inputs.pt` and `metadata.json` with `execution_status: "exception"`.

```bash
export SGLANG_KERNEL_API_LOGLEVEL=10
export SGLANG_KERNEL_API_DUMP_DIR=/tmp/kernel_dumps
```

**Note**: Tensor dumps are skipped during CUDA graph capture (copying tensors to CPU is not allowed during capture). Disable CUDA graph for debug sessions requiring level 10 dumps.

---

## 2. Common CUDA Errors and Diagnostics

| Error | Log Level | What to Check | Common Causes |
|-------|-----------|--------------|---------------|
| Illegal memory access / device-side assert | 3 | Tensor shapes, dtypes, device placement, contiguity, strides | Shape mismatch, CPU tensor to GPU kernel, wrong stride |
| NaN / Inf | 5 | min, max, mean, nan_count, inf_count | Division by zero, numerical overflow, uninitialized memory |
| CUDA OOM | 3 | Tensor shapes — are they unexpectedly large? | Wrong batch size, sequence length, accidental dimension expansion |
| Wrong dtype | 3 | dtype field in metadata | float16 vs bfloat16 mismatch |

### Crash Point Identification

Compare the last successful and first failed API calls in the log:
- **Last successful call**: has both inputs AND outputs logged
- **First failed call**: has inputs logged but NO outputs — this is where the crash occurred

---

## 3. Multi-Process Debugging

When running with multiple GPUs/processes, use the `%i` pattern for per-process logs:

```bash
export FLASHINFER_LOGLEVEL=3
export FLASHINFER_LOGDEST=debug_rank_%i.log

torchrun --nproc_per_node=4 my_script.py
```

For level 10 dumps with multi-process, also separate dump directories:

```bash
export SGLANG_KERNEL_API_DUMP_DIR=/tmp/kernel_dumps_%i
```

---

## 4. compute-sanitizer Integration

### Memory Checker

```bash
compute-sanitizer --tool memcheck python script.py
```

Combine with API logging — sanitizer shows the failing kernel, API log shows the inputs:

```bash
export FLASHINFER_LOGLEVEL=3
export FLASHINFER_LOGDEST=debug.log
compute-sanitizer --tool memcheck python script.py
```

Typical output:
```
========= Invalid __global__ write of size 4 bytes
=========     at 0x1234 in SomeKernel
=========     by thread (256,0,0) in block (10,0,0)
=========     Address 0x... is out of bounds
```

### Race Checker

```bash
compute-sanitizer --tool=racecheck python script.py
```

**Note**: `--racecheck-memcpy-async=no` controls older `cp.async` (sm80), NOT `cp.async.bulk`. False positives with `cp.async.bulk` (raw-address TMA) are a known issue — switching to descriptor-based TMA (`cp.async.bulk.tensor`) eliminates them.

---

## 5. cuda-gdb Integration

```bash
cuda-gdb --args python script.py
```

In gdb:
```
(cuda-gdb) run
(cuda-gdb) where     # Stack trace when it crashes
```

Correlate the backtrace with API log files to see what inputs caused the crash.

---

## 6. Kernel printf Debugging

### Basic Usage

```cpp
__global__ void MyKernel(const float* input, float* output, int n) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;

    if (threadIdx.x == 0 && blockIdx.x == 0) {
        printf("n=%d, input[0]=%f\n", n, input[0]);
    }

    if (idx < n) {
        output[idx] = input[idx] * 2.0f;
    }
}
```

**Always flush after kernel launch:**
```python
my_kernel(input, output)
torch.cuda.synchronize()  # Flushes printf output
```

### Choosing the Right Print Thread

| Kernel Type | Print Condition | Notes |
|-------------|----------------|-------|
| Simple kernel | `threadIdx.x == 0` | One thread per block |
| Warp-specialized | `threadIdx.x % 32 == 0` | One thread per warp |
| Group-specialized | Per-group representative | Depends on kernel's scheduling layout |

**Common mistake**:
```cpp
// Only warp 0 will print — misses all other warps!
if (threadIdx.x == 0) {
    printf("warp=%d\n", threadIdx.x / 32);
}
```

### Other Kernel Debugging Tools

```cpp
assert(value >= 0.0f && "Value must be non-negative");
static_assert(BLOCK_SIZE % 32 == 0, "BLOCK_SIZE must be multiple of warp size");
```

---

## 7. Best Practices

1. **Start with level 3** — enough for shape/dtype/device issues without overwhelming output
2. **Use level 5 only for NaN/Inf** — adds statistics computation overhead
3. **Log to file for crashes** — console output is lost when the process aborts
4. **Compare before/after** — last successful API call (has outputs) vs first failed (no outputs) = crash point
5. **Disable logging in production** — zero overhead when disabled (decorator returns original function)
6. **Disable CUDA graph for debug** — level 10 tensor dumps require CPU copies not allowed during graph capture

---

## 8. Environment Variables Quick Reference

General patterns (framework-specific env var names vary):

| Variable Pattern | Values | Description |
|-----------------|--------|-------------|
| `*_LOGLEVEL` | 0/1/3/5/10 | API logging verbosity |
| `*_LOGDEST` | stdout/stderr/path/%i | Log destination |
| `*_DUMP_DIR` | path | Directory for level-10 tensor dumps |
| `*_DUMP_INCLUDE` | wildcard | Only dump matching API names |
| `*_DUMP_EXCLUDE` | wildcard | Skip matching API names |
| `CUDA_LAUNCH_BLOCKING` | 1 | Synchronous CUDA execution (slow, for debugging) |
| `TORCH_DISTRIBUTED_DEBUG` | DETAIL | Detailed distributed logging |
| `NCCL_DEBUG` | INFO | NCCL communication logging |
| `NCCL_DEBUG_SUBSYS` | ALL | All NCCL subsystems |

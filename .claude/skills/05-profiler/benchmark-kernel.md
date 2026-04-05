# Kernel Benchmarking

<!-- source: flashinfer, sglang, Liger-Kernel -->

Accurate GPU kernel benchmarking methodology for AI infrastructure projects.

## Recommended Benchmark Design

Treat benchmarking as an extension of testing, not as an isolated timing script.

The strongest pattern here comes from Liger-Kernel:

- reuse reference implementations from tests as benchmark baselines
- keep one shared setup path so speed and memory benchmarks exercise the same inputs
- benchmark `forward`, `backward`, and `full` separately when the kernel supports them
- measure speed and memory in separate passes
- report quantiles or median plus spread, not only one average number
- keep tensor creation and module initialization outside the measured region whenever possible

This is the default best practice for new kernel work in this repo.

## Hardware Spec Prerequisite

Before interpreting kernel numbers, confirm the hardware reference you are using.

At minimum, record:

- GPU family and compute capability
- relevant Tensor Core or dense-compute peak for the active precision path
- memory bandwidth
- interconnect assumptions if the result is multi-GPU
- software support status if the hardware is new

Without that, MFU and roofline-style conclusions can be misleading.

See:

- `knowledge/hardware/gpu-spec-reference.md`
- `knowledge/hardware/b300-blackwell-notes.md`
- `knowledge/hardware/nvidia-datacenter-gpu-matrix.md`

## Timing Methods

Two primary methods for measuring kernel execution time:

<!-- source: flashinfer -->

### 1. CUPTI (Preferred)

Hardware-level profiling for the most accurate GPU kernel time. Measures pure GPU compute time without host-device overhead.

```bash
pip install -U cupti-python  # Requires CUDA 13+
```

### 2. CUDA Events (Fallback)

Standard CUDA event timing. Good accuracy with slight overhead from host synchronization. Automatically used if CUPTI is not available.

| Aspect | CUPTI (Preferred) | CUDA Events (Fallback) |
|--------|-------------------|------------------------|
| **Accuracy** | Highest (hardware-level) | Good (slight overhead) |
| **Installation** | `pip install cupti-python` | Built-in with CUDA |
| **Requirements** | CUDA 13+ | Any CUDA version |
| **Best for** | Accurate micro-benchmarks | General benchmarking |

## Python API: bench_gpu_time()

<!-- source: flashinfer -->

```python
import torch
from <library>.testing import bench_gpu_time

# Setup kernel
def my_kernel_wrapper(q, k, v):
    return some_kernel(q, k, v)

# Create test inputs
device = torch.device("cuda")
q = torch.randn(32, 8, 128, dtype=torch.bfloat16, device=device)
k = torch.randn(2048, 8, 128, dtype=torch.bfloat16, device=device)
v = torch.randn(2048, 8, 128, dtype=torch.bfloat16, device=device)

# Benchmark - CUPTI preferred, CUDA events if unavailable
median_time, std_time = bench_gpu_time(
    my_kernel_wrapper,
    args=(q, k, v),
    enable_cupti=True,          # Prefer CUPTI, fallback to CUDA events
    num_iters=30,               # Number of iterations
    dry_run_iters=5,            # Warmup iterations
)

print(f"Kernel time: {median_time:.3f} ms +/- {std_time:.3f} ms")

# Calculate FLOPS
flops = ...  # Your FLOP count
tflops = (flops / 1e12) / (median_time / 1000)
print(f"Achieved: {tflops:.2f} TFLOPS/sec")
```

## Liger-Style Benchmark Patterns

### 1. Reuse the test baseline

Do not write a second reference implementation just for benchmarking. Import the same
baseline used by correctness tests so benchmark and parity logic cannot drift apart.

Recommended rule:

- test suite owns the reference implementation
- benchmark suite imports that reference implementation
- new kernel variants compete against the exact same baseline used in correctness checks

### 2. Use one shared setup factory

Build tensors and layers through one setup function, then reuse it for both speed and
memory measurement.

Generic shape:

```python
def setup_case(cfg, provider):
    # create tensors, choose baseline or optimized implementation
    return inputs, op

def bench_speed(cfg, provider):
    inputs, op = setup_case(cfg, provider)
    return do_speed_bench(lambda: op(*inputs))

def bench_memory(cfg, provider):
    inputs, op = setup_case(cfg, provider)
    return do_memory_bench(lambda: op(*inputs))
```

This prevents "speed bench uses one setup, memory bench uses another" drift.

### 3. Separate operation modes

If the kernel meaningfully supports different execution paths, benchmark them
independently:

- `forward`
- `backward`
- `full`
- optional `no-grad-forward`

This is especially important for training kernels where forward speed and backward memory
behavior differ materially.

### 4. Keep setup out of the timing window

Unless you are explicitly measuring end-to-end user-visible latency, do not include:

- tensor allocation
- module construction
- graph compilation
- random input generation

inside the timed lambda.

Measure the kernel path, not benchmark harness noise.

### 5. Report spread, not just one scalar

Recommended outputs:

- p50 or median latency
- spread such as std or p20/p80
- throughput metrics such as TFLOPS or TB/s when meaningful
- peak memory in a separate benchmark pass

This helps distinguish true regressions from noisy runs.

### 6. Sweep safe, realistic parameter grids

Use shared config helpers or explicit sweep configs instead of ad hoc shapes.

Good sweep dimensions:

- sequence length
- hidden size or head dimension
- batch or token count
- dtype
- provider or backend

Prefer realistic model configs over arbitrary synthetic dimensions when the kernel is
meant for production workloads.

### 7. Make reproductions cheap

A strong benchmark harness should be able to emit or preserve:

- the provider name
- the tested shape or config
- operation mode
- dtype
- exact repro command

If a performance result cannot be reproduced from the saved output, the benchmark is not
yet good enough.

### Advanced Options

```python
# Cold L2 cache benchmarking
median_time, std_time = bench_gpu_time(
    my_kernel, args=(x, y),
    enable_cupti=True,
    cold_l2_cache=True,         # Flush L2 or rotate buffers
    num_iters=30
)

# Force CUDA events (skip CUPTI even if installed)
median_time, std_time = bench_gpu_time(
    my_kernel, args=(x, y),
    enable_cupti=False,
    num_iters=30
)
```

## CLI Benchmark Tool

<!-- source: flashinfer -->

For systematic benchmarking with multiple backends:

```bash
python benchmarks/benchmark.py \
    --routine <kernel_name> \
    --backends backend1 backend2 backend3 \
    --batch_size 32 \
    --num_iters 30 \
    --dry_run_iters 5 \
    --refcheck \
    -vv \
    --generate_repro_command
```

### Key Output Metrics

```
[PERF] backend_a :: median time 0.145 ms; std 0.002 ms; achieved tflops 125.3 TFLOPs/sec; achieved tb_per_sec 1.87 TB/sec
[PERF] backend_b :: median time 0.138 ms; std 0.001 ms; achieved tflops 131.5 TFLOPs/sec; achieved tb_per_sec 1.96 TB/sec
```

- **median time**: Median kernel execution time (lower is better)
- **std**: Standard deviation (lower means more consistent)
- **achieved tflops**: Effective TFLOPS throughput
- **achieved tb_per_sec**: Memory bandwidth utilization

### Batch Testing with Test Lists

Create a test list file `benchmarks.txt`:

```bash
--routine KernelA --backends impl1 impl2 --batch_size 32 --head_dim 128
--routine KernelA --backends impl1 impl2 --batch_size 64 --head_dim 128
--routine KernelB --backends impl1 impl3 --batch_size 256 --m 1 --n 1024 --k 7168
```

Run all tests:

```bash
python benchmarks/benchmark.py \
    --testlist benchmarks.txt \
    --output_path results.csv \
    --generate_repro_command \
    --refcheck
```

### Common Flags

| Flag | Description | Default |
|------|-------------|---------|
| `--num_iters` | Measurement iterations | 30 |
| `--dry_run_iters` | Warmup iterations | 5 |
| `--refcheck` | Verify output correctness | False |
| `--allow_output_mismatch` | Continue on mismatch | False |
| `--use_cuda_events` | Force CUDA events (skip CUPTI) | False |
| `--no_cuda_graph` | Disable CUDA graph | False |
| `-vv` | Very verbose output | - |
| `--generate_repro_command` | Print reproducer command | False |
| `--output_path` | Save results to CSV | None |

## Accuracy and Parity Rules

Benchmarking should not silently relax correctness.

Recommended practice:

1. run reference checking before or alongside performance measurement
1. compare against the same baseline used in tests
1. validate both output parity and gradient parity for training kernels
1. make tolerance explicit per dtype
1. if mismatch is allowed for exploratory runs, label the result clearly as non-parity-clean

For training kernels, parity usually means:

- forward output close to baseline
- backward gradients close to baseline
- no hidden dtype changes in accumulation or reduction paths

Suggested tolerance policy:

| dtype | default rtol | default atol | Notes |
|------|--------------|--------------|------|
| `float32` | `1e-5` | `1e-6` | use strict baseline parity |
| `float16` | `1e-3` to `1e-2` | `1e-3` to `1e-2` | choose based on reduction sensitivity |
| `bfloat16` | `1e-3` to `1e-2` | `1e-3` to `1e-2` | watch large-reduction kernels carefully |

If a kernel needs looser tolerances, record why. Do not silently widen them.

## E2E Server Profiling

<!-- source: sglang -->

For profiling inference server end-to-end performance:

### Workflow

1. **Launch server**:
   ```bash
   CUDA_VISIBLE_DEVICES=<gpu_id> <server> serve --model-path <model> --port <port> &
   ```

2. **Wait for readiness**:
   ```bash
   for i in $(seq 1 120); do
     if curl -s http://127.0.0.1:<port>/health 2>/dev/null | grep -q "ok\|healthy"; then
       echo "Server ready"; break
     fi
     sleep 5
   done
   ```

3. **Validate accuracy** (sanity check):
   ```bash
   python3 -m <server>.test.few_shot_gsm8k --num-q 20
   # Expected accuracy: > 0.8 for capable models
   ```

4. **Generate profile**:
   ```bash
   python3 -m <server>.test.send_one --profile
   # Output: Chrome trace at /tmp/<timestamp>/*.trace.json.gz
   ```

   Optional flags:
   - `--profile-steps N` - number of profiling steps (default: 5)
   - `--profile-by-stage` - profile by stage (prefill/decode separately)
   - `--profile-prefix <path>` - custom output prefix

5. **Cleanup**:
   ```bash
   pkill -9 -f "<server_process_pattern>"
   ```

6. **View the trace**:
   - **Perfetto UI**: https://ui.perfetto.dev/ (drag and drop)
   - **Chrome tracing**: `chrome://tracing` (load file)

## Auto-Benchmark Workflow

<!-- source: sglang -->

For systematic server-level performance tuning with search:

### Key Concepts

- Start from a baseline configuration
- Define a `search_space` of server flags to sweep
- Use tiered search (Tier 1 = smoke test, Tier 2 = recommended default, Tier 3 = exhaustive)
- Benchmark against target SLA (`max_ttft_ms`, `max_tpot_ms`)

### Search Tiers

| Tier | Description | When to Use |
|------|-------------|-------------|
| 1 | Smallest sweep, one-at-a-time changes | Smoke tests, config validation |
| 2 | Balanced coverage (recommended default) | Everyday tuning |
| 3 | Full cartesian product | Tightly bounded search spaces only |

### Important Tunable Groups

- **Kernel/backend**: attention_backend, prefill/decode attention backends, sampling_backend
- **Batching/scheduling**: max_running_requests, chunked_prefill_size, schedule_conservativeness
- **Memory/cache**: max_total_tokens, page_size, radix_cache settings
- **Parallel/distributed**: tp_size, pp_size, dp_size, ep_size
- **CUDA graph**: cuda_graph_max_bs, disable_cuda_graph_padding

## Troubleshooting

### Inconsistent Results

1. **Increase warmup**: `--dry_run_iters 10`
2. **Increase iterations**: `--num_iters 50`
3. **Use cold L2 cache**: `bench_gpu_time(..., cold_l2_cache=True)`
4. **Disable GPU boost**: `sudo nvidia-smi -lgc <base_clock>`

### Reference Check Failures

- `--allow_output_mismatch` to continue benchmarking
- Check numerical tolerance differences between backends (FP32 vs FP16)
- Use `-vv` for detailed tensor statistics

## Best Practices

1. **Install CUPTI** for best accuracy (but not required)
2. **Use reference checking** (`--refcheck`) to verify correctness alongside performance
3. **Run sufficient iterations** (30+ measurement, 5+ warmup) for statistical significance
4. **Save results to CSV** for later analysis and comparison
5. **Generate reproducer commands** for sharing and reproducibility
6. **Compare multiple backends** to find the optimal implementation
7. **Use real workload data** for production tuning, not just synthetic benchmarks
8. **Import the baseline from tests** instead of maintaining a separate benchmark reference
9. **Separate speed and memory passes** so peak-memory probing does not contaminate timing
10. **Benchmark forward/backward/full independently** for training kernels

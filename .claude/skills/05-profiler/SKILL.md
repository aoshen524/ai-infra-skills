---
name: profiler
description: Performance profiling, benchmarking, and NCU analysis for AI infrastructure
model: claude-opus-4-6
tools: [Bash, Read, Grep, Glob]
---

# Profiler & Benchmarking Skill

<!-- source: flashinfer, sglang, flash-attention -->

Performance profiling, kernel benchmarking, and low-level analysis for AI infrastructure projects.

## Scope

This skill covers three areas:

1. **Kernel Benchmarking** (`benchmark-kernel.md`) - Accurate GPU kernel timing using CUPTI/CUDA Events, batch testing, backend comparison, and result export.

2. **Performance Tuning** (`performance-tuning.md`) - Block size tuning, occupancy analysis, SASS disassembly for MMA instructions, and optimization strategies for SM90+ architectures.

3. **E2E Profiling** - Server-level profiling with Chrome/Perfetto traces for inference engines.

## Quick Reference

### Kernel Benchmarking

```bash
# Time a kernel with CUPTI (most accurate)
pip install -U cupti-python

# Python API
from <library>.testing import bench_gpu_time
median_ms, std_ms = bench_gpu_time(my_kernel, args=(x, y), enable_cupti=True, num_iters=30)
```

### E2E Profiling (Inference Servers)

```bash
# Launch server, validate, capture trace
<server> serve --model-path <model> --port 30000 &
# Wait for ready, then trigger profiling
<profiler_client> --profile --profile-steps 5
# Output: Chrome trace at /tmp/<timestamp>/*.trace.json.gz
# View in: https://ui.perfetto.dev/ or chrome://tracing
```

### NCU / Nsight Analysis

```bash
# Profile a specific kernel
ncu --set full -o profile python my_script.py

# Quick occupancy check
ncu --metrics sm__warps_active.avg.pct_of_peak_sustained_active python my_script.py
```

## Files

- `benchmark-kernel.md` - Kernel benchmarking methodology and tools
- `performance-tuning.md` - Block size tuning, SASS analysis, and optimization

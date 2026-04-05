<!-- source: ricardojacomini/b300-benchmarks -->

# B300 and Blackwell Bring-Up Notes

## What This Covers

This document captures the parts of the NVIDIA B300 benchmark suite that are most useful
for AI infra reasoning.

It is not a complete product spec sheet. It is a bring-up reference for:

- microbenchmark interpretation
- interconnect validation
- software maturity on new hardware
- benchmark layering from kernel to end-to-end training

Source repo:

- `ricardojacomini/b300-benchmarks`

## Why This Repo Is Useful

The strongest pattern in `b300-benchmarks` is not one result table. It is the structure:

- microbenchmarks for compute and bandwidth
- P2P and collective validation
- end-to-end training benchmark
- environment pinning
- report-driven root-cause analysis

That is exactly the right shape for early hardware bring-up notes.

## Key Hardware Dimensions to Preserve

### Identity

The repo treats B300 as:

- NVIDIA B300 SXM6
- Blackwell family
- SM `10.3`

Those identifiers matter because compiler and runtime support often gate on arch support
long before the rest of the stack is polished.

### Benchmark Layers

The repo explicitly covers:

- GPU microbenchmarks: GEMM, attention, conv2d, memory bandwidth, NVLink all-reduce
- P2P validation: raw link bandwidth, streamed full-mesh, NCCL-style collective
- end-to-end training: Pangu S2S

This layering is important because it prevents people from explaining every shortfall
with one benchmark category alone.

### Software Maturity

One of the most important findings in the repo is that early nightly or CUDA support for
new hardware may still fall back in practice.

The useful engineering takeaway is:

- "nightly supports the new GPU" is not enough
- check actual arch lists, compiler target support, and runtime behavior
- record the exact build that was used
- pin the environment for reproducibility

The repo explicitly records exact nightly builds and warns that new-hardware support may
still use fallback cubins or unsupported compile paths.

### Interconnect Validation

The repo does not stop at "NVLink exists". It validates:

- hardware link bandwidth
- streamed full-mesh P2P behavior
- NCCL-style collective behavior

That is the correct way to reason about multi-GPU systems. A nominal interconnect spec is
not enough for distributed AI infra decisions.

## Best Practices to Reuse

### 1. Record exact environments

For early hardware, record:

- exact framework build
- exact CUDA version
- whether stable, nightly, or container image was used
- why that choice was necessary

### 2. Separate theoretical capability from current usability

A hardware family may advertise support for a feature while the actual toolchain still
falls back.

Keep two separate statements:

- what the hardware is supposed to enable
- what the current software stack actually enables

### 3. Benchmark across layers

Use all three:

- micro
- communication
- end-to-end

Without that split, you cannot tell whether the bottleneck is kernel, interconnect, or
framework maturity.

### 4. Treat new hardware bring-up as a moving target

For new GPUs, the knowledge note should always preserve:

- observation date
- exact build or container
- known missing support
- expected inflection point for maturity

This is more useful than a timeless-looking static spec table.

## How This Should Influence Our Repo

For `ai-infra-skills`, hardware references should:

- include concrete GPU-family notes when they materially affect engineering decisions
- explicitly separate hardware potential from software readiness
- connect hardware notes to kernel, distributed, and serving experts
- preserve benchmark surface coverage, not only peak numbers

## Review Questions

1. are we recording actual software support status, not only hardware claims?
1. do we have at least one benchmark per layer: compute, interconnect, and end-to-end?
1. are exact nightly or container builds pinned when new hardware support is immature?
1. do we explain fallback behavior clearly enough for engineers to interpret results?

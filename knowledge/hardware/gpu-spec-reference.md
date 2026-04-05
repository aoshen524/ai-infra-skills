<!-- source: b300-benchmarks, common AI infra practice -->

# GPU Spec Reference

## What This Covers

This document defines what hardware facts should be captured in `ai-infra-skills` for a
GPU family or accelerator platform.

The goal is not to collect marketing tables. The goal is to preserve the hardware facts
that materially change AI infra decisions:

- kernel design
- MFU or roofline interpretation
- bandwidth expectations
- distributed topology choices
- software-stack compatibility

## Why This Matters

Hardware sits at the bottom of the AI infra stack. If the hardware model is wrong, all
higher-level conclusions become noisy:

- a kernel may look slow only because the peak denominator is wrong
- a training job may under-scale because the interconnect assumption is wrong
- a compiler or runtime may fall back because the software stack does not actually target
  the device

For that reason, hardware references belong in the knowledge base, not only in ad hoc
benchmark notebooks.

## What To Record for Each GPU Family

### 1. Identity

Capture:

- product name
- form factor such as SXM, PCIe, OAM
- architecture family
- SM or compute capability
- typical deployment shape such as 1-GPU workstation, 8-GPU node, or NVLink domain

Why:

- many toolchains key behavior off architecture and form factor, not only marketing name

### 2. Compute Envelope

Capture:

- relevant dense compute peaks
- Tensor Core peaks by precision when available
- which precision is meaningful for the workloads you care about
- whether reported numbers are theoretical peak or measured benchmark ceilings

Why:

- MFU and roofline calculations are only useful when the chosen denominator matches the
  actual precision path

### 3. Memory System

Capture:

- HBM or GDDR capacity
- memory bandwidth
- important cache or shared-memory constraints when they materially affect kernel design

Why:

- many attention and communication paths are memory-bound before they are compute-bound

### 4. Interconnect

Capture:

- NVLink generation or equivalent fabric
- PCIe generation when relevant
- peer-to-peer support assumptions
- practical collectives or P2P bandwidth findings if available

Why:

- distributed performance depends on the real topology, not the nominal GPU count

### 5. Software Support Status

Capture:

- minimum CUDA version that matters in practice
- stable vs nightly framework status
- compiler gaps
- runtime fallback behavior
- missing cubin or arch-target support

Why:

- new hardware is often blocked more by software maturity than by hardware capability

### 6. Benchmark Surfaces

Capture at least one representative benchmark for each layer:

- microbenchmarks
- kernel-level or operator-level benchmarks
- collective or P2P tests
- end-to-end training or inference benchmarks

Why:

- the useful question is not "what is the peak?" but "which layer is currently limiting us?"

## Recommended Per-GPU Reference Template

```text
GPU family
  identity
  compute envelope
  memory system
  interconnect
  software support status
  benchmark surfaces
  caveats and current fallbacks
```

## How To Use This in Practice

Concrete family notes should sit next to this template. In this repo, the main NVIDIA
companion is:

- `knowledge/hardware/nvidia-datacenter-gpu-matrix.md`

### Kernel work

Use the hardware reference before:

- choosing the MFU denominator
- deciding whether a kernel is compute-bound or memory-bound
- assuming a Tensor Core or compiler path exists

### Distributed work

Use the hardware reference before:

- interpreting NCCL or P2P results
- designing tensor or data parallel layouts
- assuming peer access or NVLink behavior

### Serving and system work

Use the hardware reference before:

- deciding whether nightly builds are acceptable
- interpreting runtime fallback behavior
- making capacity or topology claims

## Review Questions

1. does this hardware note record facts that change engineering decisions, not just specs for their own sake?
1. are compute, memory, interconnect, and software support all represented?
1. is there at least one benchmark surface recorded beyond theoretical peak?
1. does the note explain current fallback behavior on immature software stacks?

<!-- sources: NVIDIA A100 product page, NVIDIA H100 product page, NVIDIA H200 product page, NVIDIA HGX platform page, NVIDIA Blackwell Tuning Guide, NVIDIA DGX B300 User Guide -->

# NVIDIA Data Center GPU Matrix

## What This Covers

This note is the concrete companion to `gpu-spec-reference.md`.

It summarizes the NVIDIA data center GPU families that show up most often in AI infra
work today:

- A100
- H100
- H200
- B200
- B300

The goal is not to mirror every datasheet row. The goal is to preserve the hardware
differences that actually change engineering decisions for:

- kernel benchmarking and MFU interpretation
- distributed topology assumptions
- long-context or memory-bound workload planning
- new-hardware bring-up risk

## Reading Rules

### Do not mix SKU scopes

NVIDIA publishes some families as single-GPU product pages and others as HGX-platform
tables. That means:

- A100, H100, and H200 numbers below are mostly single-GPU signals
- B200 and B300 numbers are often published at the HGX 8-GPU node level
- when a B200 or B300 per-GPU quantity is inferred from the HGX total, this document
  says so explicitly

### Do not use one family name as a denominator

Before drawing MFU, roofline, or scaling conclusions, pin the exact deployment shape:

- PCIe vs SXM
- single GPU vs 8-GPU HGX domain
- current CUDA, driver, and framework build

This matters especially for H100 and Blackwell-family systems.

## Quick Routing

Use this note to decide where to look next:

- if the question is mostly memory capacity or long context, start with H200 or B300
- if the question is mature training or inference baselines, start with A100 or H100
- if the question is toolchain readiness, cubins, PTX, or new-arch support, jump to the
  Blackwell notes for B200 and B300
- if the question is node-local scaling behavior, pay attention to NVLink generation and
  whether the published number is per GPU or per HGX domain

## Generation Matrix

| Family | Architecture signal | Memory signal | Interconnect signal | Software signal | Good default use |
| ------ | ------------------- | ------------- | ------------------- | --------------- | ---------------- |
| `A100` | Ampere, `sm80` class | 80GB HBM2e, about 2.0 TB/s on SXM | 3rd-gen NVLink, 600 GB/s | most mature baseline in many training stacks | reproducible baseline, compatibility anchor, older-cluster expectation setting |
| `H100` | Hopper, `sm90` class | 80GB HBM3 on SXM, 94GB on NVL; bandwidth depends on SKU and reaches 3.35 TB/s on SXM | 4th-gen NVLink, up to 900 GB/s on SXM | mature in current CUDA and framework stacks, but exact SKU still matters | main Hopper baseline for kernel and distributed work |
| `H200` | Hopper, `sm90` class | 141GB HBM3e, 4.8 TB/s | Hopper-era scale-out story, typically same decision surface as H100 but with a much stronger memory system | usually lower bring-up risk than Blackwell while solving many memory-pressure cases | long-context serving, memory-bound kernels, larger batch windows |
| `B200` | Blackwell, `sm100` class | HGX B200 publishes 1.4 TB total HBM3e across 8 GPUs, about 192 GB per GPU if divided evenly | 5th-gen NVLink, 1.8 TB/s GPU-to-GPU, 14.4 TB/s total NVLink bandwidth on HGX B200 | version-sensitive: verify driver, CUDA, PTX, cubin, and framework arch targeting before trusting numbers | Blackwell bring-up, FP4 or FP8 exploration, bandwidth-heavy multi-GPU nodes |
| `B300` | Blackwell Ultra generation | HGX B300 publishes up to 2.1 TB total memory across 8 GPUs, about 288 GB per GPU if divided evenly | 5th-gen NVLink, 1.8 TB/s GPU-to-GPU, 14.4 TB/s total NVLink bandwidth on HGX B300 | newest and most version-sensitive path; treat all performance claims as environment-specific until validated locally | very large memory footprints, newest-cluster benchmarking, Blackwell Ultra readiness work |

## Engineering Takeaways by Family

### A100

A100 remains the cleanest compatibility baseline for many teams:

- broad framework support
- well-understood `sm80` kernel behavior
- mature MIG support
- stable denominator for "is the new stack obviously broken?"

Use A100 when you want a conservative baseline before attributing regressions to a newer
architecture.

### H100

H100 is the default Hopper-era baseline, but SKU details matter:

- SXM and NVL have different memory and bandwidth envelopes
- interconnect differs across SXM HGX systems and PCIe or NVL deployments
- some kernel claims that are true on H100 SXM do not transfer cleanly to H100 NVL

When a benchmark result is being used to justify a kernel redesign, record the H100 SKU
explicitly.

### H200

H200 is usually the first answer when the problem is memory rather than pure compute:

- 141GB HBM3e and 4.8 TB/s materially change long-context and KV-heavy workloads
- it stays in the Hopper family, so the software-risk profile is usually easier than a
  fresh Blackwell bring-up
- many "we need Blackwell" requests are actually "we need more memory bandwidth or more
  HBM capacity" requests, which H200 may already solve

For serving and attention-heavy work, H200 is often the simplest way to relieve memory
pressure without paying the full new-architecture tax.

### B200

B200 changes both kernel and system assumptions:

- Blackwell is a new architecture generation, so toolchains must target it correctly
- HGX B200 exposes a much faster NVLink domain than Hopper-era baselines
- public numbers are often platform-level, so engineers must avoid accidental apples-to-
  oranges comparisons against single-GPU Hopper or Ampere figures

Use B200 when the work is explicitly about Blackwell-class capability, not just "a fast
GPU."

### B300

B300 should be treated as a bring-up and validation target first, and a timeless
baseline second:

- the memory footprint is large enough to change workload partitioning decisions
- Blackwell Ultra platform tables are strong for node-level planning
- software maturity, arch targeting, and benchmark methodology matter as much as the raw
  hardware table

Pair this note with `b300-blackwell-notes.md` whenever the question is actual observed
performance on new Blackwell Ultra systems.

## Blackwell-Specific Build and Compatibility Rules

The most important Blackwell software rule is simple:

- do not assume an existing Hopper binary is a valid Blackwell benchmark

NVIDIA's compatibility guidance for Blackwell says:

- Blackwell is compute capability `10.0`
- binaries that include PTX can JIT forward more safely
- binaries that ship only older cubins may need rebuilds
- arch-conditional targets such as `sm_100a` are not generic forward-compatible fallbacks

For repo usage, that means:

1. Record exact CUDA and driver versions for B200 or B300 runs.
1. Preserve the actual `-gencode` or build-system arch list.
1. Treat "runs successfully" and "uses the intended kernel path" as different questions.
1. Re-check cubin, PTX, and fallback behavior before publishing any MFU claim.

## What To Record In Benchmark Notes

When a result references one of these GPUs, capture at least:

- exact SKU or platform shape
- single-GPU or HGX-domain scope
- memory capacity and memory bandwidth used in interpretation
- NVLink generation and whether the result depends on it
- CUDA or framework version if the GPU is Blackwell-family

## Default Recommendations for This Repo

Use these defaults unless the task says otherwise:

- use `A100` when you need a mature compatibility baseline
- use `H100` as the mainstream Hopper baseline
- use `H200` when the bottleneck is memory capacity or memory bandwidth
- use `B200` and `B300` only with explicit environment capture and architecture validation
- if the result is multi-GPU, document the NVLink domain before discussing scaling quality

## Adjacent References

- `knowledge/hardware/gpu-spec-reference.md`
- `knowledge/hardware/b300-blackwell-notes.md`
- `knowledge/distributed/nccl-patterns.md`
- `knowledge/kernels/kernel-autoresearch-loop.md`

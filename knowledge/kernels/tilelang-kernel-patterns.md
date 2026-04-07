# TileLang Kernel Patterns

Use this reference when the implementation surface is explicitly TileLang rather than raw
CUDA, Triton, or an existing framework-specific custom-op stack.

## What TileLang Is Good For

TileLang is most useful when you need:

- a Python DSL for custom GPU kernels
- explicit control over tiles, shared memory, and register fragments
- cross-vendor portability across CUDA and HIP-class targets
- a faster path to GEMM-, attention-, or MLA-like kernels than handwritten CUDA

It is not the default answer for every kernel problem. If the target project already has a
stable CUDA, Triton, or CUTLASS path, extend that path first before introducing a second
kernel stack.

## Development Contract

Before writing code, lock:

- operator shape and dtype contract
- target hardware family
- expected baseline for parity and speed
- benchmark shape that will stay frozen during optimization

For TileLang work, the fastest way to get lost is to change the kernel and benchmark
shape at the same time.

## Core Building Blocks

TileLang kernels usually rely on five ideas:

1. `T.Kernel(...)` defines grid shape and thread count
1. `T.alloc_shared(...)` reserves shared memory tiles
1. `T.alloc_fragment(...)` reserves register-backed fragments
1. `T.copy(...)` moves data across memory levels
1. `T.gemm(...)` maps matrix math to the best available MMA path

Shared memory should usually be annotated with swizzled layouts for matrix-heavy kernels.
Skipping swizzle is one of the easiest ways to create bank-conflict regressions.

## Recommended Kernel Loop

1. define the tensor contract and output surface
1. choose initial tile sizes that fit the target hardware
1. allocate shared memory and register fragments
1. apply swizzle layouts to the shared-memory operands
1. pipeline the K-loop with `T.Pipelined(...)`
1. validate against a trusted reference
1. benchmark and only then optimize

For most GEMM-like kernels, a solid first pass is:

- FP16 or BF16 inputs
- FP32 accumulators
- `block_K` aligned to tensor-core-friendly sizes
- two- or three-stage pipelining

## Common Patterns

### GEMM

Strong default for GEMM-style kernels:

- shared-memory A and B tiles
- FP32 accumulator fragment
- swizzled shared layout
- pipelined K-loop

Tune block sizes per architecture instead of assuming one universal tile.

### FlashAttention-Style Kernels

For attention-style kernels:

- separate score and output accumulator fragments
- keep online reduction state explicit
- verify the stage split before tuning occupancy
- benchmark with realistic sequence and head shapes

### MLA and Narrow-Matrix Paths

For MLA-like or narrow-matrix kernels:

- watch warp policy and operand orientation
- confirm whether the chosen layout still produces efficient MMA shapes
- re-check parity across odd head-dim or latent-dim cases

These kernels often fail because a tile choice that looks natural algebraically creates an
unfriendly memory or MMA layout on real hardware.

## Hardware Guidance

### NVIDIA

- `A100`: prefer conservative shared-memory usage and three-stage pipelines
- `H100`: larger tiles may fit, but only keep them if the register budget survives
- `B200` or `B300`: treat compiler maturity and generated code shape as first-class risks

### AMD or Other Targets

- reduce assumptions about shared-memory capacity and warp semantics
- re-validate tile sizes and launch geometry per backend
- do not assume a CUDA-tuned tile transfers cleanly

## Validation Discipline

TileLang kernel work should always ship with:

- parity against one canonical reference path
- dtype-explicit tolerances
- edge-shape coverage for tails and non-divisible dimensions
- a benchmark that reuses the same logical input contract as the tests

If the benchmark uses a different setup path from correctness, speedups become harder to
trust.

## Debugging Checklist

If a TileLang kernel fails:

- check shared-memory budget first
- verify matrix dimensions at every `T.gemm(...)`
- inspect boundary handling for tail tiles
- confirm shared-memory layouts were annotated as intended
- reduce to one tile and one launch block before escalating

If performance is poor:

- look for missing swizzle
- compare two-stage versus three-stage pipelining
- re-evaluate tile sizes against shared-memory and register budgets
- validate that the generated kernel is still the real hot path in the integrated system

## Relationship To Other Repo Guides

Use this document with:

- `.claude/skills/03-design/tilelang-kernel.md` for TileLang-specific implementation flow
- `.claude/skills/03-design/add-kernel.md` for generic kernel-integration structure
- `.claude/skills/05-profiler/benchmark-kernel.md` for benchmark methodology
- `knowledge/kernels/sass-mma-analysis.md` when generated MMA behavior must be verified

## Review Questions

1. is TileLang actually the right kernel surface for this project?
1. is the benchmark harness frozen enough to trust optimization results?
1. are shared-memory layout and pipeline depth explicit instead of accidental?
1. does the kernel still match the integrated serving or training path after optimization?

# TileLang Kernel Development

Use this guide when the user specifically wants a TileLang implementation or the target
repo already uses TileLang as the kernel surface.

## When To Use

Use it for:

- new TileLang GEMM, attention, or MLA kernels
- TileLang kernel optimization
- TileLang compilation or runtime debugging
- portability questions across CUDA and HIP-class targets

Do not use it when:

- the project already has a stable non-TileLang kernel surface that should be extended
- the task is only about CUDA launch glue, FFI, or framework registration
- benchmarking or profiling is the only missing piece

## Workflow

1. lock the tensor contract, target hardware, and baseline
1. read `knowledge/kernels/tilelang-kernel-patterns.md`
1. sketch the kernel around `T.Kernel`, shared tiles, fragments, and one pipelined loop
1. validate parity against a trusted reference
1. benchmark with the repo's normal harness
1. only then tune tiles, stages, or warp policy

## Core Implementation Defaults

- use shared-memory tiles for matrix operands
- use FP32 accumulators unless the user explicitly accepts weaker numerics
- apply swizzled shared-memory layouts for matrix-heavy paths
- choose tensor-core-friendly K blocking
- keep one benchmark harness fixed across optimization rounds

## What To Check First When Things Go Wrong

- shared-memory budget
- `T.gemm(...)` operand dimensions
- tail-tile and boundary handling
- missing layout annotation
- whether the generated TileLang path is actually the integrated hot path

## Use With

- `knowledge/kernels/tilelang-kernel-patterns.md`
- `.claude/skills/03-design/add-kernel.md`
- `.claude/skills/05-profiler/benchmark-kernel.md`
- `.claude/skills/05-profiler/performance-tuning.md`

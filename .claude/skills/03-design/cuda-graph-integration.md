# CUDA Graph Integration

Use this guide when the user wants to add or debug CUDA Graph support in a serving or
inference stack.

## When To Use

Use it for:

- adding CUDA Graph to a decode or steady-state serving path
- stabilizing an existing graph integration
- deciding graph granularity or bucket strategy
- combining CUDA Graph with `torch.compile`

Do not use it for:

- generic kernel benchmarking with no graph integration
- distributed hang debugging
- training-loop-only dynamic systems that clearly lack a stable repeated path

## Workflow

1. identify the true steady-state repeated path
1. read `knowledge/serving/cuda-graph-patterns.md`
1. remove or isolate host-device sync from the hot path
1. preallocate persistent GPU buffers for graph-read inputs
1. warm up until lazy init, compilation, and cache setup are stable
1. capture one stable bucket or stage first
1. validate replay correctness before scaling to more buckets
1. compare eager, compile-only, graph-only, and combined modes when relevant

## Key Decisions

### Capture Timing

Capture after the runtime path is stable, not during setup.

### Buffer Strategy

Prefer persistent buffers over per-step allocation.

### Graph Granularity

Choose the smallest graph surface that still covers the real repeated work.

### Bucket Strategy

Use multiple buckets only when one graph cannot fairly cover the dynamic workload.

## Use With

- `knowledge/serving/cuda-graph-patterns.md`
- `knowledge/serving/torch-compile-coverage.md`
- `.claude/skills/05-profiler/torch-profile-triage.md`
- `.claude/agents/serving-systems-expert.md`

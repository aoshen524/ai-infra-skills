<!-- source: Awesome-ML-SYS-Tutorial CUDA Graph article -->

# CUDA Graph Patterns for Serving Systems

Use this document when a serving or inference path is fast enough on GPU kernels but still
pays too much CPU launch overhead, or when a runtime wants to stabilize a repeated decode
path under `CUDA Graph`.

## Why CUDA Graph Matters

CUDA Graph is valuable when the runtime repeatedly launches the same GPU work shape.

In serving systems, the main win is simple:

- capture a repeated kernel sequence once
- instantiate it into an executable graph
- replay it with one CPU launch instead of many

That is why CUDA Graph often matters more in steady-state inference than in general
training loops.

## Mental Model

Think in three phases:

### Capture

The runtime records kernel launches, memory operations, and stream dependencies into a DAG.

### Instantiate

The recorded DAG is compiled into an executable plan. At this point:

- dependency structure is fixed
- unsupported operations fail validation
- kernel parameters and tensor addresses are effectively baked into the executable graph

### Replay

The runtime submits the instantiated graph with one launch. GPU-side execution still
follows the internal DAG, but CPU overhead drops sharply because the dispatcher is no
longer issuing every kernel individually.

## Core Constraints

CUDA Graph is powerful because it is static. The same reason also makes it fragile.

### 1. Pointer Stability

Replay assumes the GPU virtual addresses used during capture are still valid.

If the tensor address changes, the captured graph can read or write the wrong memory.

### 2. No Dynamic Allocation Inside Capture

Tensors needed by the graph should be preallocated before capture.

If the hot path creates new tensors during capture, allocator behavior can break address
stability and invalidate replay assumptions.

### 3. No Host-Device Synchronization

Graph capture should not cross host-side sync boundaries.

Common offenders:

- `.item()`
- sync-heavy sampling paths
- control logic that pulls values back to CPU mid-step

### 4. Static Control Flow

The captured graph represents one fixed computation shape.

If loop counts, branches, or shape-driven code paths vary at runtime, that usually implies:

- one graph per stable case
- or fallback to eager execution

### 5. Code-Path Changes Require Re-Capture

A recorded graph does not update itself when Python or framework code changes.

If the path changes materially, capture again.

## Batch Shape Implication

A single graph usually serves one stable batch or shape bucket.

In decode systems, that often means:

- capture a small set of batch-size or token-shape buckets
- replay when the request matches one of them
- otherwise fall back to eager mode

This is often a better engineering choice than trying to force one graph to cover every
dynamic case.

## Memory Pool Model

CUDA Graph is best understood as a specialized cache with a memory cost.

The graph needs a stable memory region for its internal tensors. Two important ideas
follow:

### Peak Matters More Than Sum

The graph footprint is closer to the high-water mark during capture than to the sum of
every intermediate tensor ever created.

### Graphs Can Share A Pool

Multiple graphs do not always need separate isolated peak allocations.

If only one graph is replayed at a time, several batch buckets can often reuse one shared
graph memory pool instead of multiplying peak memory linearly.

This is frequently the difference between a practical multi-bucket graph system and an
unusable one.

## Persistent Buffer Pattern

One of the most useful serving patterns is the persistent-buffer contract:

- allocate stable GPU buffers outside the graph
- write new values into those buffers between replays
- let the captured graph read from the same stable addresses every time

This preserves pointer stability while still allowing dynamic per-step content.

Use this pattern for:

- decode inputs
- cached intermediate state
- small changing metadata that must still live on GPU

Without a pattern like this, engineers often reintroduce allocation churn and accidentally
break graph safety.

## Deferred Capture

Do not capture too early.

Many systems need one warmup pass or one-time initialization before the true steady-state
path exists. Examples:

- lazy kernel compilation
- lazy module setup
- runtime cache initialization
- shape-dependent one-time setup

Deferred capture means:

- run enough warmup to stabilize the real path first
- only then capture the steady-state path

This prevents half-initialized graphs that miss important work or capture setup-only logic.

## Single Graph vs Multi Graph

There is no universal answer.

### Single Graph

Prefer one graph when:

- the path is structurally uniform
- one graph can cover the stable steady-state work
- simplicity and maintainability matter most

### Multi Graph

Prefer multiple graphs when:

- batch sizes or stages differ materially
- one static graph would force too many fallbacks
- separate buckets or stages have clean stable boundaries

The real tradeoff is not only performance. It is also:

- memory footprint
- bucket-management complexity
- fallback behavior
- maintenance burden

## Multi-Stage Or Unified Coverage

Some systems have multiple repeated subpaths that look different at first glance but can
still be unified under one graph strategy if:

- their state can be stabilized
- their buffers can be made persistent
- the control path is static enough after warmup

This matters for asymmetric or staged autoregressive systems where "one model step" is
really composed of several tightly repeated sub-forwards.

The right question is not "do we have multiple submodels?" but:

- can the steady-state repeated path be made static enough for one capture domain?

## CUDA Graph And `torch.compile`

These two systems can help each other, but they can also conflict.

Useful mindset:

- `torch.compile` reduces Python overhead and reshapes the compute region
- CUDA Graph removes repeated CPU launch overhead for the stabilized region

Before combining them, verify:

- the compiled path is already stable enough
- graph capture is occurring after the compile path is settled
- graph and compile ownership of the runtime path do not silently fight each other

If the combined system is slower or more fragile, inspect capture timing and region shape
before blaming one kernel.

## Integration Workflow

1. identify the real steady-state serving path
1. remove sync-heavy operations or move them outside capture
1. preallocate persistent buffers
1. warm up until the runtime path is stable
1. capture one stable bucket or stage
1. replay and compare against eager execution
1. extend to additional buckets only when the first bucket is clearly correct

## Common Failure Modes

| Symptom | Likely Cause | First Fix |
| ------- | ------------ | --------- |
| replay crashes or corrupts outputs | tensor addresses changed | move inputs to persistent buffers and re-capture |
| capture fails unexpectedly | sync or unsupported op in the path | remove host-device sync from the hot path |
| graph helps one batch size but not others | one graph was stretched across dynamic shapes | bucket batch sizes and keep eager fallback |
| memory cost explodes with many graphs | each graph has isolated peak memory | use a shared graph pool when replay is mutually exclusive |
| graph misses important work | capture happened before lazy init or warmup settled | defer capture until steady state |
| compile plus graph gets unstable | ownership of the stabilized region is unclear | validate compile region first, then graph the steady-state path |

## Relationship To Other Repo Guides

Use this document with:

- `.claude/skills/03-design/cuda-graph-integration.md`
- `knowledge/serving/torch-compile-coverage.md`
- `.claude/skills/05-profiler/torch-profile-triage.md`
- `.claude/agents/serving-systems-expert.md`

## Review Questions

1. is the target path actually steady-state and repetitive enough for CUDA Graph?
1. are all graph-read buffers pointer-stable across replays?
1. did capture happen after lazy init and warmup?
1. is the system using the smallest graph set that still covers the real workload?

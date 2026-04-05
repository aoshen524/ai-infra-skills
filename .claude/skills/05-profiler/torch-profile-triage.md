# Torch Profile Triage

<!-- source: SGLang torch profiler analysis article -->

Use this guide when a server trace needs to be translated into concrete optimization
actions, not just opened in Perfetto and stared at.

## What This Is For

This workflow turns raw torch profiler traces into three outputs:

- where the GPU time went
- which kernels still have overlap headroom
- which existing fusion or overlap patterns should be reused first

## When to Use

Use it for:

- inference-server trace triage
- prefill or decode bottleneck analysis
- overlap and fusion opportunity ranking
- mapping GPU kernels back to Python scopes

Do not use it for:

- kernel microbenchmarks
- distributed hangs without a valid trace
- "open Perfetto and browse manually" as the only workflow

## Two-Phase Trace Collection

The strongest pattern is to collect two different traces for two different purposes.

### Phase 1: Mapping Trace

Collect a trace with graph optimizations relaxed enough to recover mappings from:

- GPU kernel
- CPU op
- Python scope

Use this trace to answer: "what code path launched this kernel?"

### Phase 2: Formal Trace

Collect the real serving trace with the production-like execution mode enabled.

Use this trace to answer: "how much overlap exists in the actual runtime shape?"

Do not try to force one trace to answer both questions equally well.

## Stage-Aware Profiling

Prefer profiling by stage:

- extend or prefill
- decode

If the serving stack uses prefill and decode disaggregation, collect stage-specific
traces from the relevant workers instead of merging everything into one ambiguous trace.

## Default Triage Outputs

Every serious trace triage should emit three tables.

### 1. Kernel Table

Required fields:

- stage
- kernel
- time share
- mapped Python location when available

### 2. Overlap Opportunity Table

Required fields:

- stage
- kernel
- Python scope
- priority
- recommendation

This table should distinguish between:

- kernels with real remaining overlap headroom
- kernels already hidden under other compute

### 3. Fuse Opportunity Table

Required fields:

- stage
- pattern
- current location
- candidate fused path

Do not stop at "this kernel is hot." The useful answer is often "this hot path matches a
known fusion or overlap pattern we already understand."

## Recommended Analysis Order

1. map kernels back to Python scopes
1. separate prefill from decode
1. rank by time share
1. check overlap headroom
1. compare against the existing fusion or overlap catalog
1. only then propose new optimization work

This order avoids reinventing optimizations the project already has.

## Perfetto Is Not The Only Output Surface

A good triage flow should also produce compact text outputs:

- small tables
- actionable rows
- optional ASCII timeline summary

That lets engineers judge the trace before opening a heavy UI.

## Common Failure Modes

| Symptom | Likely Cause | First Fix |
| ------- | ------------ | --------- |
| hot kernels are visible but unmapped | mapping trace was skipped | collect the graph-relaxed mapping trace first |
| overlap conclusions are wrong | only graph-off trace was used | compare against the formal graph-on trace |
| fusion suggestion is noisy | no catalog comparison was done | match against known fuse or overlap patterns first |
| one trace is unreadable | prefill and decode were mixed together | split collection by serving stage |

## Review Questions

1. were both mapping and formal traces collected?
1. was the trace split by stage when the runtime has distinct stages?
1. does the output include kernel, overlap, and fusion views?
1. are proposed optimizations tied back to a concrete Python scope?

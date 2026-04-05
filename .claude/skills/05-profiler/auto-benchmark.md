# Auto-Driven Benchmarking

<!-- source: SGLang auto-driven benchmark article -->

Use this guide when benchmark work is really a search problem over server flags,
workloads, and SLA constraints rather than a single fixed command.

## What This Is For

This workflow turns benchmark tuning into a resumable experiment loop:

1. canonicalize the workload
1. generate candidate launch configs
1. run the service and load
1. judge against SLA
1. save results in machine-readable form
1. resume later instead of starting from zero

## When to Use

Use it for:

- server flag search
- backend comparison on one workload family
- QPS and concurrency tuning
- speculative decoding or staged serving search
- repeated benchmark sweeps that otherwise become shell-script sprawl

Do not use it for:

- one-off kernel microbenchmarks
- profiling a single already-fixed server config
- benchmarking with undefined workload shape

## Required Interfaces

A healthy auto-benchmark workflow exposes three entrypoints:

- `convert`: normalize input data into one canonical benchmark format
- `validate`: catch dataset shape issues before the expensive run
- `run`: execute the search or sweep

This separation prevents expensive benchmark time from being wasted on malformed input.

## Canonical Dataset Rule

Convert every source workload into one benchmark format before search begins.

Typical sources:

- ShareGPT-style JSON
- custom JSONL
- synthetic random prompts
- generated shared-prefix cases

Artifacts to keep:

- `prepared_dataset.jsonl`
- validation output or schema result
- scenario name and backend choice

## Candidate Search Rules

Treat launch flags as a structured search space, not a hand-maintained shell history.

Useful dimensions include:

- `tp`, `dp`, `pp`, `ep`
- chunked prefill sizes
- scheduler or memory flags
- speculative decoding flags
- QPS or concurrency limits

Recommended search modes:

- `tier 1`: smallest sanity sweep
- `tier 2`: balanced default search
- `tier 3`: slow and broad exploration

If the space becomes too large, cap it with a `max_candidates` style limit instead of
letting the search explode.

## SLA-Driven Execution

A benchmark candidate is not "best" just because throughput is high.

Record both throughput and latency:

- TTFT
- TPOT
- output tokens per second
- concurrency
- whether the SLA passed

Support both:

- fixed-QPS sweeps
- QPS binary search with lower, upper, and tolerance bounds

## Resume and Live Results Are Mandatory

Long-running benchmark searches should survive interruption.

Persist:

- `live_results.jsonl` during execution
- `results.jsonl` at scenario completion
- `results.csv` for fast comparison
- `summary.md` for quick reading

If the process is interrupted, the next run should load finished trials and continue.

## Multi-Scenario Expansion

One config should often be benchmarked against multiple workload scenarios.

Good examples:

- chat
- summarization
- long-context decode
- shared-prefix batch

Avoid declaring victory from one toy input distribution.

## Suggested Output Table

Keep a compact summary per candidate:

| Candidate | Stage | QPS | Output tok/s | TTFT ms | TPOT ms | SLA |
| --------- | ----- | --- | ------------ | ------- | ------- | --- |
| 0 | base | 1.0 | 51.8 | 6.3 | 1.2 | pass |

The exact columns can differ, but the run should always preserve:

- launch arguments
- workload scenario
- backend
- throughput
- latency
- pass or fail decision

## Two-Stage Search Pattern

For advanced serving stacks, split search into:

- base stage
- speculative or optional advanced stage

Only run the expensive second stage after a sane base candidate exists.

## Common Failure Modes

| Symptom | Likely Cause | First Fix |
| ------- | ------------ | --------- |
| benchmark crashes after a long startup | dataset shape was invalid | require `validate` before `run` |
| "best" result is not reproducible | run config was not serialized | save the exact candidate flags and scenario |
| search takes too long | candidate space is too large | reduce tier or set `max_candidates` |
| results are lost after interruption | live artifacts were not written incrementally | write `live_results.jsonl` on every completed trial |

## Review Questions

1. is the workload canonicalized before search starts?
1. does the benchmark save partial progress during the run?
1. are SLA decisions recorded alongside throughput metrics?
1. is the search space explicit and bounded?

<!-- source: Zhihu article "如何用Opus 4.6写出一个比开源社区更快的gpu算子", sparse-mask-attention -->

# Kernel Autoresearch Loop

## What This Covers

This document captures a high-leverage workflow for iterative GPU kernel optimization
with an AI coding agent.

It is not about one specific optimization trick. It is about building a closed loop
where the agent can safely improve a kernel through repeated measurable iterations.

## Core Idea

The key pattern is:

```text
frozen evaluation harness
    -> one small kernel change
    -> correctness check
    -> performance check
    -> profiler or PTX analysis
    -> log result
    -> next round
```

This works well because kernel work has unusually strong feedback:

- correctness is testable
- speed is measurable
- profiler evidence is concrete
- regressions are obvious

## Minimal Repo Shape

A strong kernel-autoresearch repo usually separates:

- `csrc/` or equivalent mutable kernel code
- one unified entrypoint such as `run.py`
- a short rule file such as `optimization_rules.md`
- a per-round log such as `perf_log.md`

Recommended split:

- the agent may edit only kernel implementation files by default
- the benchmark and correctness harness should be treated as mostly immutable
- the rules file and perf log become the resume surface for new sessions

## The Most Important Guardrail

Do not let the optimization agent freely edit the evaluation harness once the harness is
trusted.

Why:

- changing the benchmark case can create fake speedups
- changing correctness logic can hide regressions
- changing comparison baselines invalidates longitudinal results

Default policy:

- kernel code is mutable
- correctness and perf harness is read-only unless a human confirms a real bug in it

## Round Structure

Each optimization round should be small and attributable.

Recommended rules:

1. make one meaningful change per round
1. run correctness before performance
1. print or save the measured result every round
1. log the change, reason, and outcome in `perf_log.md`
1. keep successful rounds easy to resume from

This prevents "big-bang" edits where no one knows which idea actually mattered.

## Correctness Contract

Kernel optimization should stay attached to a fixed correctness contract:

- a canonical reference implementation
- explicit tolerance by dtype
- stable representative shapes
- forward parity
- backward parity when the kernel is used in training

If correctness cases drift every round, the optimization loop stops being trustworthy.

## Performance Contract

The performance side should also be fixed enough to compare across rounds:

- representative benchmark shape
- stable dtype
- stable device
- stable baseline implementations
- fixed throughput or MFU formula

Good loops often have multiple evaluation modes:

- `correctness`
- `perf`
- `compare`

The same entrypoint should expose all three so the agent does not invent a different
testing path in every round.

## Escalation Ladder

When straightforward changes stop helping, escalate the evidence source rather than
guessing harder.

Recommended ladder:

1. benchmark result
1. correctness diff
1. profiler output
1. PTX or SASS inspection
1. targeted external search for missing optimization ideas

Do not jump to random rewrites before exhausting the stronger evidence sources.

## Plateau Handling

A useful rule is to treat repeated no-gain rounds as a signal to change strategy.

Example trigger:

- after 3 rounds with no meaningful gain, require one of:
  - profiler-guided diagnosis
  - PTX or SASS inspection
  - external search for known optimization patterns

This helps the loop escape local minima instead of endlessly polishing small details.

## Context Management

Long optimization sessions degrade model quality. Resume should come from artifacts, not
from keeping giant chat history alive.

Recommended resume artifacts:

- `optimization_rules.md`
- `perf_log.md`
- the current kernel file
- the unified benchmark harness

This makes it cheap to clear context and restart with a fresh model session.

## What Makes This Work

The workflow succeeds when all four are true:

- the goal is measurable
- the mutable surface is constrained
- each iteration is small
- evidence is written down between rounds

Without those conditions, autoresearch turns into prompt churn instead of engineering.

## Review Questions

1. is the benchmark and correctness harness frozen enough to trust longitudinal results?
1. is each round small enough that gains or regressions are attributable?
1. does the agent have profiler or PTX evidence before major rewrites?
1. can a fresh session resume from logs without relying on long hidden context?

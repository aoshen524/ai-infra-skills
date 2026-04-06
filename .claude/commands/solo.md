---
name: solo
description: Run a repo-agnostic autonomous implementation or experiment loop with persistent state and keep/discard discipline.
allowed-tools: [Bash, Read, Grep, Glob]
---

<!-- source: generalized from existing solo workflow and autoresearch -->

# Solo

Run a self-directed loop that plans, executes, verifies, records, and iterates until the
task reaches a stop condition or the user interrupts it.

`/solo` is a command orchestrator. It should route into the right expert, knowledge, and
skill documents when domain-specific judgment is needed instead of duplicating those
documents inline.

## Usage

```text
/solo [task goal]
```

## Command Goal

Accept the user's goal, then lock a session contract before starting autonomous work.

The session contract must define:

- task goal
- acceptance criteria
- repo path, working tree policy, and branch strategy
- mutable surface
- frozen surface
- run command, check command, and metric extraction command
- metric direction: `higher is better` or `lower is better`
- timeout per run
- max iterations
- stop conditions

If any required field is missing, fill it in from repo context when reasonable. If the
missing field materially changes risk or evaluation fairness, stop and ask the user for
that specific missing detail before entering the loop.

## Modes

`/solo` supports exactly two modes.

### Implementation Mode

Use for:

- multi-file implementation
- bug fixes
- feature delivery
- code changes followed by validation

Shape:

1. lock the session contract
1. inspect the relevant code and choose the right expert or skill
1. implement the smallest sound change set
1. run checks and validation commands
1. record findings and outcomes
1. iterate until the acceptance criteria or stop conditions are reached

### Experiment Mode

Use for:

- autotuning
- benchmark-driven optimization
- training configuration search
- serving flag or runtime parameter search
- any fixed-harness keep/discard loop

Shape:

1. lock the session contract
1. freeze the evaluation harness and metric extraction path
1. run baseline first
1. test one clear hypothesis at a time
1. keep improved states and discard non-improved states
1. persist every result in the session ledger
1. continue until stop conditions or user interruption

Do not create a second `/experiment-loop` command. Experiment semantics live here.

## Session Persistence

By default, create `solo_session/` in the active repo or worktree. It should contain:

- `task_plan.md`
- `findings.md`
- `progress.md`
- `results.tsv`

`results.tsv` does not need to be committed to git unless the user explicitly wants that.

### File Responsibilities

`task_plan.md`

- task goal
- acceptance criteria
- chosen mode
- session contract
- iteration table

`findings.md`

- key discoveries
- decisions and reversals
- failure modes
- reasons for keeping or discarding candidate changes

`progress.md`

- time-ordered execution log
- commands run
- checkpoints
- recovery notes after crash or timeout

`results.tsv`

- one row per experimental iteration
- minimum columns:
  - `iteration`
  - `commit_or_working_state`
  - `primary_metric`
  - `secondary_metric_or_resource`
  - `status`
  - `short_description`

Use `status` values: `keep`, `discard`, or `crash`.

## Hard Operating Rules

### Baseline First

Before any experiment loop, run one baseline with the fixed harness and record it. Do not
evaluate candidate changes without a baseline from the same environment and measurement
method.

### Mutable vs Frozen Surface

Define these before editing:

- `mutable surface`: files, modules, flags, or parameters that may change
- `frozen surface`: evaluation harness, datasets, benchmarks, result extraction logic,
  external constraints, and any other fairness-critical pieces

Default rule: harness, benchmark shape, data, and metric extraction logic stay frozen.

### One Clear Hypothesis Per Iteration

Each experiment iteration should try one main idea. Avoid changing multiple variables at
once unless the task explicitly requires a coupled change.

### Keep or Discard

If the change improves the primary metric while still meeting acceptance constraints, keep
it.

If the change does not improve the outcome, revert to the iteration starting point and log
the result as `discard`.

If the change crashes:

- if the failure is easy and local, fix it and retry within the same idea
- if the idea itself is poor or unstable, log `crash`, revert, and move on

### Simplicity Criterion

Do not keep small gains that introduce disproportionate complexity. Prefer:

- equal or better results with simpler code
- a slightly smaller gain with a much cleaner design
- stable changes over brittle improvements

### Timeout and Recovery

Every run must have a timeout.

If a run times out:

- treat it as a failure for the iteration
- record the timeout in `progress.md` and `results.tsv`
- decide whether it was an environment issue or an idea issue before retrying

If the environment is busy, already running jobs, or missing prerequisites, stop first and
surface the blocker. Do not blindly steal GPUs, kill unrelated jobs, or mutate shared
state just to continue the loop.

### Log Hygiene

Redirect long command output to files by default. Read back only the key lines needed for
decision-making with targeted tools such as `grep`, `rg`, or `tail`.

Do not flood model context with:

- full benchmark logs
- full compile logs
- full training logs
- full profiler traces

Capture only the metrics, error signatures, and short excerpts needed to make the next
decision.

### Long-Loop Autonomy

Once the session contract is locked and the loop begins, continue autonomously until:

- the acceptance criteria are met
- the max iteration count is reached
- a hard blocker prevents fair progress
- the user interrupts the session

Do not repeatedly ask whether to continue after every iteration.

## Standard Workflow

### Phase 1: Lock the Session Contract

Capture:

- goal
- mode
- acceptance criteria
- repo path and branch strategy
- mutable and frozen surfaces
- run, check, and metric extraction commands
- timeout
- max iterations
- stop conditions

Then initialize `solo_session/` and record the contract in `task_plan.md`.

### Phase 2: Read Before Acting

Read the relevant source files before editing or running experiments. Also route to the
closest expert or workflow when the domain is specialized.

Examples:

- distributed issue -> `distributed-systems-expert`
- kernel issue -> `kernel-performance-expert`
- serving compile regression -> `knowledge/serving/torch-compile-coverage.md`
- benchmark method -> `.claude/skills/05-profiler/*`

`/solo` should orchestrate these resources, not replace them.

### Phase 3: Execute the Loop

For implementation mode:

1. inspect code and form a minimal change plan
1. make the change
1. run the relevant checks
1. record outcomes and next decision

For experiment mode:

1. run baseline first
1. choose one hypothesis
1. apply the candidate change within the mutable surface
1. run with timeout and log redirection
1. extract metrics using the frozen extraction method
1. write the result to `results.tsv`
1. keep or discard the state

### Phase 4: Final Report

Finish with a short report that includes:

- original goal
- chosen mode
- iteration count
- retained changes
- key metrics and comparisons
- important findings
- blockers or risks
- recommended next steps

## Session Contract Template

Use a structure equivalent to this before starting the loop:

```text
Goal:
Acceptance criteria:
Mode: implementation | experiment
Repo/worktree/branch strategy:
Mutable surface:
Frozen surface:
Run command:
Check command:
Metric extraction command:
Metric direction:
Timeout:
Max iterations:
Stop conditions:
```

## Results Ledger Template

Use a tab-separated ledger with at least:

```text
iteration	commit_or_working_state	primary_metric	secondary_metric_or_resource	status	short_description
```

## Quality Bar

The command should remain:

- repo-agnostic
- fair in how it compares experiments
- explicit about mutable versus frozen surfaces
- durable across long loops because state lives on disk
- disciplined about keep/discard/revert decisions

Another agent should not need to invent baseline, results ledger, crash handling, or
revert rules after reading this command.

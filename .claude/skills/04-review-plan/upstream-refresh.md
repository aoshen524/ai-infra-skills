# Upstream Refresh

Use this guide when the goal is to absorb the latest high-value progress from the tracked
upstream repositories into `ai-infra-skills`.

This is a maintenance and synthesis workflow, not a blind mirroring workflow.

## When To Use

Use it for:

- periodic upstream audits
- "what changed recently in the tracked repos?"
- updating this repo after upstream framework or workflow evolution
- checking whether our current skills, knowledge, and experts are still aligned

Do not use it for:

- answering one-off framework questions
- implementing a normal feature in a downstream codebase
- copying large amounts of upstream documentation without synthesis

## Tracked Upstreams

This workflow tracks 15 upstream repositories:

1. `AReaL`
1. `Megatron-LM`
1. `TensorRT-LLM`
1. `flash-attention`
1. `flashinfer`
1. `nccl`
1. `ollama`
1. `OpenRLHF`
1. `sglang`
1. `slime`
1. `torchtitan`
1. `XTuner`
1. `verl`
1. `vllm`
1. `yzlnew/infra-skills`

Treat the first 14 as the main framework set and `yzlnew/infra-skills` as the additional
upstream skills source that now feeds this repo.

## Audit Contract

Before starting, lock:

- audit window
  - for example: releases and merged changes since the last audit date
- evidence sources
  - official repos, release notes, docs, merged PRs, and important issue discussions
- target output surface
  - `knowledge`, `skills`, `agents`, `commands`, `rules`, `README`, `CLAUDE`, or audit docs
- acceptance standard
  - what counts as "worth absorbing" versus "watch only"

If the user did not specify the audit window, default to "since the last recorded audit or
the last major repo update, whichever is easier to verify."

## Core Rules

### 1. Evidence First

Every upstream claim should be backed by:

- repo name
- concrete date
- link or file path
- short note on why it matters

Do not update this repo based on vague memory or social summary alone.

### 2. Prefer Durable Changes

Absorb changes that affect one or more of:

- public workflow shape
- debug method
- benchmark methodology
- API or config contracts
- architecture decisions
- repeated failure modes
- agent routing or maintenance ergonomics

Usually do not absorb:

- cosmetic refactors
- one-off implementation churn with no durable lesson
- niche internal details with no reuse value

### 3. Synthesize, Do Not Mirror

Never dump upstream docs into this repo verbatim.

Instead:

- extract the stable pattern
- generalize away repo-local naming when possible
- preserve crucial framework-specific identifiers only when they are needed for retrieval

### 4. Update The Right Layer

Route each absorbed change to the correct layer:

- `knowledge/*` for stable reference and design context
- `.claude/skills/*` for repeatable workflows
- `.claude/agents/*` for domain-routing changes
- `.claude/commands/*` for reusable command orchestration
- `.claude/rules/*` for durable constraints
- `INDEX.md`, `KNOWLEDGE_INDEX.md`, `EXPERT_KNOWLEDGE_MAP.md`, `CLAUDE.md`, `README.md`, `COVERAGE_AUDIT.md` for navigation and audit state

If a finding does not clearly belong to one layer, it is usually not ready to absorb.

### 5. Compare Against Existing Coverage First

Before adding a new file, check:

- do we already cover this pattern elsewhere?
- is the current coverage stale, shallow, or misplaced?
- is this an update, a new doc, or just a better index entry?

Prefer strengthening an existing canonical doc over creating a duplicate.

## Recommended Workflow

### Phase 1: Build The Audit Table

Create a simple table or working note with:

- upstream repo
- audit window
- surfaces checked
- high-signal findings
- decision: `absorb`, `watch`, or `ignore`
- target file(s) in `ai-infra-skills`

### Phase 2: Scan High-Signal Surfaces

For each upstream, prioritize:

- release notes
- docs or design docs
- merged PRs with workflow impact
- new benchmark or debug practices
- new architecture or integration patterns

Prefer primary sources over secondary commentary.

### Phase 3: Extract Candidate Lessons

For each candidate lesson, write:

- what changed
- why it matters
- which users or agents benefit
- where it belongs in this repo

If you cannot explain where it belongs, mark it `watch` instead of forcing it in.

### Phase 4: Update This Repo

Apply the minimum correct set of changes:

- update or add knowledge docs
- update skill routing or examples
- adjust expert ownership if a topic boundary changed
- update indexes and README-level references if discoverability changed

### Phase 5: Reconcile The Audit State

After edits, update the maintenance layer when needed:

- `COVERAGE_AUDIT.md`
- `README.md`
- `CLAUDE.md`
- `EXPERT_KNOWLEDGE_MAP.md`

The repo should clearly show that the new upstream knowledge was absorbed, not only hide it
inside a leaf file.

## Repo-Specific Heuristics

### `sglang`, `vllm`, `TensorRT-LLM`, `ollama`

Look for:

- serving topology changes
- benchmark or profiler workflows
- model onboarding and runtime debugging lessons
- compile-region or backend integration updates

### `AReaL`, `verl`, `slime`, `OpenRLHF`, `XTuner`

Look for:

- rollout and reward workflow changes
- agent-loop contracts
- online RL serving or gateway integration
- engine selection and backend integration guidance
- dataset and sampler contracts for large-scale training

### `Megatron-LM`, `torchtitan`

Look for:

- config and launch contract changes
- memory planning or parallelism lessons
- engine integration and checkpoint behavior
- build, test, and CI workflow changes

### `flash-attention`, `flashinfer`, `TileLang`, `nccl`

Look for:

- kernel debug methodology
- benchmark methodology
- hardware-specific optimization patterns
- JIT, codegen, or low-level integration shifts

### `yzlnew/infra-skills`

Look for:

- reusable skill patterns we do not yet expose cleanly
- repo-navigation aids for important frameworks
- maintenance-friendly scaffolds for communication or tooling

Treat it as a skills-source repo, not as a framework repo.

## Update Decision Rules

Mark a finding `absorb` when:

- it is new enough to matter
- it is durable enough to survive beyond one PR
- it clearly improves our repo's coverage or navigation

Mark it `watch` when:

- the idea is promising but immature
- the upstream change is still in flux
- we need more evidence before turning it into canonical guidance

Mark it `ignore` when:

- it has no reusable value here
- it duplicates an existing stronger pattern
- it would increase clutter more than utility

## Output Shape

A good upstream-refresh result should include:

```markdown
## Audit Window

## Repos Checked

## Findings To Absorb

## Watch List

## Repo Changes Made

## Remaining Gaps
```

## Validation Checklist

1. every absorbed finding is tied to a concrete upstream source
1. every repo change lands in the correct layer
1. indexes are updated when discoverability changed
1. duplicated or stale guidance was avoided
1. the final repo is easier to navigate than before the audit

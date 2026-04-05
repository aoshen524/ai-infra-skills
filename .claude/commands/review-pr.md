---
name: review-pr
description: Intelligent PR code review with dynamic domain-based agent allocation.
allowed-tools: [Read, Grep, Glob, Bash]
---

<!-- source: AReaL -->

# Review PR

Review the current branch PR or a specific PR number using a four-phase workflow that
adapts review depth to the domains and risks in the diff.

## Usage

```text
/review-pr [PR_NUMBER] [--quick]
```

## Workflow Overview

```text
Phase 1: PR discovery and change analysis
Phase 2: Review task planning
Phase 3: Execute domain reviews
Phase 4: Score findings and summarize
```

## Phase 1: PR Discovery and Change Analysis

1. Resolve the target PR:

```bash
gh pr view <pr_number> --json number,title,body,state,isDraft,files,commits
```

2. Stop if the PR is closed or cannot be resolved.
3. Build a change summary:
   - modified files
   - dominant domains
   - likely risk level
   - stated testing or rollout claims

Review domains often include:

- distributed runtime
- numerics and tensor semantics
- model loading and checkpointing
- kernels and backends
- infra, launchers, or CI
- docs and tests

If `--quick` is set, stop after Phase 1 and return a concise risk summary.

## Phase 2: Review Task Planning

Generate focused review tasks by risk area.

Planning rules:

1. Every PR gets at least one review task.
1. High-risk areas get dedicated tasks.
1. Merge tasks only when the same context is needed to judge them.
1. Prefer these review depths:
   - `comprehensive` for distributed, numerics, checkpointing, kernels
   - `targeted` for scoped feature work
   - `basic` for docs or low-risk hygiene

For each task, specify:

- reason
- focus files
- risks to verify
- expected evidence

## Phase 3: Execute Domain Reviews

Read the changed files plus enough surrounding context to answer:

- is the change correct in the modified code?
- does it preserve config/API contracts?
- are fallback paths still valid?
- are tests sufficient for the actual risk?

If the environment supports sub-agents, parallelize independent domain reviews. If not,
execute them sequentially but keep outputs separated by task.

### Finding Format

```text
severity: CRITICAL | HIGH | MEDIUM | LOW
confidence: 0-100
file: path/to/file
line: 123
issue: concise description
reason: why it matters
suggestion: concrete fix or validation step
```

## Phase 4: Score Findings and Summarize

Confidence guide:

- `0`: false positive or clearly pre-existing
- `25`: plausible but unverified
- `50`: likely real but limited impact
- `75`: very likely real and important
- `100`: confirmed and reproducible

Final output structure:

```markdown
# PR Review Summary

## PR Overview
- title
- domains
- risk level

## Findings
- ordered by severity, then confidence

## Open Questions
- anything that blocks full confidence

## Review Statistics
- issue counts by severity
```

## Review Rules

- Findings come first.
- Include file and line references for every finding.
- Do not claim coverage for paths you did not read.
- Treat distributed and numerical changes as higher risk than surface area alone suggests.

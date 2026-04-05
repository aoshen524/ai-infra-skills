---
name: commit
description: Generate intelligent commit messages based on staged changes using Conventional Commits format.
allowed-tools: [Bash, Read, Grep]
---

<!-- source: AReaL -->

# Commit

Generate a Conventional Commit message from staged changes, confirm it with the user, and
then create or amend the commit.

## Usage

```text
/commit [--amend] [--scope <scope>]
```

## Workflow

### 1. Analyze Staged Changes

```bash
git diff --cached --name-only
git diff --cached
git log --oneline -5
```

If nothing is staged, stop and ask the user to stage changes first.

### 2. Classify the Change

Choose the best Conventional Commit type:

| Type | Use When |
| ---- | -------- |
| `feat` | new capability |
| `fix` | bug fix |
| `docs` | documentation-only |
| `refactor` | restructuring without behavior change |
| `test` | test-only changes |
| `chore` | maintenance, deps, tooling |
| `perf` | performance improvement |

### 3. Determine Scope

Infer scope from the staged files. Examples:

- `workflow`
- `engine`
- `model`
- `kernel`
- `infra`
- `api`
- `docs`
- `tests`

If `--scope <scope>` is provided, use it unless it clearly conflicts with the diff.

### 4. Generate the Message

Format:

```text
<type>(<scope>): <subject>

<body>

Key changes:
- item 1
- item 2
```

Rules:

- imperative subject
- no period at the end of the subject
- keep the subject concise
- explain why in the body, not just what changed
- include `Key changes` only when the diff is complex enough to benefit from bullets

### 5. Confirm and Commit

Show the proposed message, then run one of:

```bash
git commit -m "<message>"
git commit --amend -m "<message>"
```

## Scope Hints

- model loading, conversion, architecture -> `model`
- distributed training, sharding, launchers -> `engine` or `infra`
- kernels, fused ops, attention backends -> `kernel`
- docs and guides -> `docs`

## Examples

```text
fix(engine): validate world size before FSDP init
docs(infra): add CI timeout triage checklist
perf(kernel): reduce shared memory pressure in attention path
```

# Remote GPU Backend Workflow

<!-- source: SGLang auto-driven development article -->

Use this guide when the coding agent runs locally but the real execution target is a
remote GPU machine.

The goal is to treat remote hosts as standardized execution backends instead of
maintaining a separate ad hoc workflow for every server.

## When to Use

Use this workflow when:

- model loading or kernel validation requires a remote GPU box
- local code changes need to be verified on H100, H200, or B200 class hardware
- the project depends on a long-lived dev container on a remote machine
- you need a repeatable way to run smoke tests, benchmarks, or server validation remotely

Do not use this workflow when:

- the task can be completed entirely on the current machine
- the remote host is not trusted or not prepared for shared development work
- the task requires cluster-wide scheduling rather than one remote backend

## Core Principle

The local agent session should stay the control plane.

The remote GPU host should be the execution plane:

- local agent owns planning, edits, and artifact interpretation
- remote host owns GPU execution, containers, and heavyweight validation

This keeps context in one place while avoiding per-machine agent drift.

## Host Profile Contract

Before using a remote backend, define a small host profile:

- host alias
- expected GPU family
- default user
- default container name
- default repo path
- cleanup preference
- common validation tasks that fit that machine

Examples:

- `h100-dev`: Hopper serving and torch.compile validation
- `h200-diffusion`: long-context or diffusion inference validation
- `b200-dev`: Blackwell bring-up, kernel, and topology-sensitive checks

## Preflight Checklist

Run these checks before syncing code or launching work:

```bash
ssh <host> 'hostname && whoami'
ssh <host> 'docker ps --format "{{.Names}}|{{.Status}}" | sed -n "1,5p"'
ssh <host> 'nvidia-smi --query-gpu=index,name,memory.used,memory.total --format=csv,noheader,nounits'
```

If a default container is expected:

```bash
ssh <host> 'docker exec <container> bash -lc "pwd && python -V && nvidia-smi -L"'
```

Also verify any repo-specific environment assumptions:

- `HF_TOKEN`
- model cache location
- framework-specific env vars
- custom kernel build artifacts

## Choose One Remote Workspace Strategy

Do not improvise a new workspace layout for every run. Pick one of these three patterns.

### 1. Reuse the remote repo directly

Use when:

- the remote repo is already clean
- you are validating changes already pushed to a branch

### 2. Create a detached remote worktree

Use when:

- the remote repo is shared
- you need branch isolation without recloning
- the machine already has a canonical checkout

### 3. Stream the local working tree to a temporary remote directory

Use when:

- you must validate uncommitted local changes
- you do not want to disturb the shared remote checkout

Generic pattern:

```bash
rsync -az --delete ./ <host>:<remote_tmp_dir>/
ssh <host> 'cd <remote_tmp_dir> && git status --short || true'
```

## Standard Remote Validation Ladder

Use a fixed validation order so failures are caught early:

1. syntax or import checks
1. lightweight unit tests
1. GPU smoke test
1. server boot validation
1. benchmark or profile collection

Typical commands:

```bash
python -m py_compile <changed_python_files>
python -m compileall <target_dir>
pytest -q <small_test_target>
CUDA_VISIBLE_DEVICES=<gpu> python <gpu_smoke_test>.py
python -m <server_entry> ...
```

## Machine-Specific Routing Rules

Treat hardware choice as part of the workflow:

- prefer `H100` for mature Hopper baselines
- prefer `H200` for memory-heavy serving or diffusion validation
- prefer `B200` or `B300` for Blackwell-specific correctness, performance, or compatibility checks

Do not ask one machine to answer a question that belongs to another architecture class.

## Safe Remote Workflow Rules

- inspect remote repo state before writing into it
- prefer temp directories or detached worktrees for uncommitted local validation
- log exact host, container, repo path, and visible GPUs in every report
- keep cleanup explicit so temporary validation directories do not accumulate silently

## Common Failure Modes

| Symptom | Likely Cause | First Fix |
| ------- | ------------ | --------- |
| remote validation differs from local | different container or dependency state | record image, env, and repo path first |
| GPU appears free but launch fails | wrong container or wrong visible devices | check `docker exec ... nvidia-smi` and `CUDA_VISIBLE_DEVICES` |
| code sync works but imports fail | host-side repo path or venv mismatch | validate inside the target container, not only on the host |
| benchmark result is noisy | machine was shared or background jobs changed load | capture `nvidia-smi` before and after the run |

## Review Questions

1. is the local session still the control plane?
1. was the remote host profile made explicit before execution?
1. was one workspace strategy chosen intentionally instead of ad hoc copying?
1. did validation climb from syntax to GPU smoke to benchmark instead of jumping straight to the slowest step?

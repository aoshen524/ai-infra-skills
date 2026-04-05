<!-- source: SGLang -->

# SGLang CI System Overview

## Scheduled Runs

The primary CI mechanism for regression detection is scheduled runs in `pr-test.yml`:

- **Trigger**: Cron schedule on `main` branch (not on PR branches)
- **Purpose**: Consistent baseline for performance and correctness tracking
- **Why scheduled**: PR-triggered runs are noisy (different base commits, variable load, sometimes skipped). Scheduled runs provide a stable time series.

```yaml
# Example cron trigger in pr-test.yml
on:
  schedule:
    - cron: '0 */6 * * *'  # Every 6 hours
  push:
    branches: [main]
```

Scheduled runs are the **primary data source** for detecting regressions. Always prefer them over PR-triggered runs when investigating.

## Test Partitioning

Tests are distributed across multiple runner types to parallelize execution:

- Tests are assigned to specific partitions (runner groups)
- Partition assignments can change over time as tests are added/removed/rebalanced
- A test that was in partition A last week might be in partition B this week
- When investigating a failure, check which partition the test currently runs in

### Why This Matters

If a test suddenly "stops running," it might have been moved to a different partition, not removed. Check the workflow file for current assignments before assuming a test was deleted.

## Runner Identification

Different runner types have different hardware, which affects test behavior:

### H200 Runners

- Path prefix: `/root/actions-runner/`
- Runner name pattern: `gpu-h200-worker-*`
- Characteristics: Hopper architecture, higher memory bandwidth, different CUDA compute capability

### Standard Runners

- Path prefix: `/public_sglang_ci/runner-*`
- Various GPU types (A100, etc.)

### Identifying Runner from Logs

```bash
# In CI logs, look for:
# 1. Runner name in the job header
# 2. nvidia-smi output at job start
# 3. Working directory path

# Using gh CLI to check a specific run
gh run view <run-id> --log | head -50
```

## Bisecting Regressions

### Always Use Scheduled Runs

```bash
# CORRECT: filter by scheduled event
gh run list --workflow pr-test.yml --event schedule --limit 20

# WRONG: includes PR-triggered runs (noisy, different conditions)
gh run list --workflow pr-test.yml --limit 20
```

PR-triggered runs are unreliable for bisection because:
- They may run on different runner types
- Base commit varies (merge base vs HEAD)
- Some PRs skip certain test partitions
- Concurrency limits may cancel runs

### Check Runner Identity First

Before concluding "commit X broke the test":

1. Verify the failing and passing runs used the **same runner type**
2. Check if the runner was swapped (e.g., A100 -> H200 migration)
3. If runners differ, it's likely a hardware/environment difference, not a code regression

```bash
# Compare runner identity across runs
gh run view <failing-run-id> --json jobs --jq '.jobs[].runnerName'
gh run view <passing-run-id> --json jobs --jq '.jobs[].runnerName'
```

## Package Version Awareness

SGLang depends on fast-moving packages that can cause regressions independent of SGLang code changes:

### Key Dependencies

- **sgl-kernel**: Custom CUDA kernels, versioned independently. A new sgl-kernel release can change behavior.
- **flashinfer-python**: Attention kernel library. Version bumps can change numerics or performance.
- **vllm**: Sometimes used as a dependency for specific features.

### Container Environment Staleness

CI runners use Docker containers that may have stale package versions:

```bash
# Check versions in CI logs
pip show sgl-kernel flashinfer-python
python -c "import sgl_kernel; print(sgl_kernel.__version__)"
```

A "regression" that appears after a container rebuild might actually be a dependency version change, not an SGLang code change.

## Key Diagnostic Patterns

### Same SHA, Different Results

**Pattern**: Same commit SHA passes on runner A but fails on runner B.

**Diagnosis**: This is almost always a hardware/environment difference, not a code bug.

**Check**:
- GPU type (A100 vs H200)
- CUDA driver version
- Container image version
- Package versions (sgl-kernel, flashinfer)

### All Runners Fail After Commit X

**Pattern**: All runner types start failing after a specific commit is merged.

**Diagnosis**: This is a genuine code regression.

**Action**:
1. Identify the commit range (last passing scheduled run -> first failing scheduled run)
2. Check the commits in that range: `git log --oneline <pass-sha>..<fail-sha>`
3. If multiple commits, bisect by checking out midpoint and running the failing test locally
4. Verify the fix by running the test before and after the revert/fix

### Flaky Test (Intermittent Failures)

**Pattern**: Same test, same runner type, passes sometimes and fails sometimes.

**Diagnosis**: Usually a timing issue, resource contention, or numerical tolerance.

**Check**:
- Is the test sensitive to GPU memory state? (Previous test left allocations)
- Does it use `pytest.approx` with tight tolerance?
- Is there a race condition in server startup/shutdown?

<!-- source: TensorRT-LLM, SGLang, torchtitan -->

# CI Failure Triage

This guide covers how to retrieve CI failures, bisect regressions, and group failures by
root cause instead of chasing symptoms.

## 1. Retrieve Failure Evidence

Start by pulling the exact failing signal from the CI system in use.

### GitHub Actions

```bash
gh run list --limit 20 --json databaseId,workflowName,headSha,conclusion,createdAt
gh run view <run_id> --json jobs
gh run view <run_id> --job <job_id> --log
```

### Jenkins

Retrieve the build URL from PR comments, bots, or job links, then inspect `testReport`
or equivalent job artifacts:

```bash
curl -s "<jenkins_job_url>/testReport/api/json"
```

### GitLab

Use pipeline, bridge, and job APIs to follow the failure chain until you reach the
terminal jobs with raw traces.

### Retrieval Rules

1. Capture the first specific error, not only the final wrapper exception.
1. Preserve run ID, job ID, commit SHA, and failing test or workload name.
1. Read raw logs for at least one representative failure in each tentative bucket.

## 2. Bisect a CI Regression

Use this six-phase method when the failure appears to be a true regression.

### Phase 1: Capture the Failure Signature

Record:

- exact test or workload
- first causal error
- whether it is deterministic
- relevant numeric values, model/config, and runner info

### Phase 2: Find the Pass/Fail Boundary

Identify:

- last known passing run
- first known failing run
- candidate commits between them

### Phase 3: Check Runner and Environment Differences

Compare:

- GPU type
- driver/runtime version
- container image or package versions
- job partition or runner pool

If the same commit passes on one runner class and fails on another, treat environment or
hardware differences as first-class hypotheses.

### Phase 4: Reproduce Locally or Remotely

Use the smallest reproducer possible:

- single test
- reduced model
- smaller batch or sequence length
- single-GPU or single-node control run

### Phase 5: Run `git bisect`

Use a test command judged only by exit code:

```bash
git bisect start
git bisect bad <bad_ref>
git bisect good <good_ref>
```

At each step:

- build if required
- run the test command
- mark good/bad based on exit code only

### Phase 6: Report

Report:

- first bad commit
- evidence linking that commit to the failure
- whether the issue is code regression, environment shift, hardware-specific behavior, or flakiness

## 3. Pipeline Failure Analysis by Root-Cause Bucket

Every failed job should end up in an explicit bucket. Avoid a final "misc" bucket.

### Recommended Buckets

| Bucket | Examples |
| ------ | -------- |
| `infra/resource` | OOM, disk full, quota exhausted |
| `infra/runtime` | timeout, node preemption, cancelled job |
| `env/dependency` | missing package, incompatible version, bad image |
| `config/schema` | missing key, invalid flag, schema drift |
| `distributed/sync` | NCCL hang, rank mismatch, world-size mismatch |
| `numerics/semantics` | dtype, mask, loss, parity, tolerance regressions |
| `model/artifact` | checkpoint not found, bad tokenizer, missing weights |
| `backend/compiler` | kernel compile failure, unsupported arch, graph capture issues |

### Bucketing Rules

Group failures together only if:

- they share the same first causal signature
- they likely belong to the same subsystem
- one coherent fix could address all of them

Split failures when:

- the first causal error differs
- one looks like infra noise and another looks like a code bug
- the likely fixes would touch unrelated files or owners

## 4. Key Pattern Diagnosis Table

| Pattern | Most Likely Diagnosis | First Check |
| ------- | --------------------- | ----------- |
| Same SHA passes on one runner, fails on another | runner or hardware-specific issue | compare GPU, driver, image, and job pool |
| Failure appears after one narrow commit range | code regression | inspect changed files and bisect |
| Same failure text across many jobs | shared subsystem bug | bucket by first causal stack frame |
| Generic wrapper exception at the end of logs | downstream symptom only | scan upward for first specific error |
| Timeout only on large world size | distributed scaling issue | compare rendezvous, collectives, and startup time |
| OOM only after a config change | config or batch-size regression | diff resolved runtime config |
| Single flaky pass among many fails | unstable test or race | compare seed, runner, timing, and retries |

## Triage Principles

1. Logs first, assumptions second.
1. Use the earliest specific failure as the anchor.
1. Separate code regressions from environment drift before proposing fixes.
1. If a bucket is real but under-validated, open an issue instead of forcing a PR.

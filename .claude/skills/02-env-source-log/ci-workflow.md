# CI/CD Workflow Patterns

Synthesized patterns for CI pipeline design, fast-fail mechanisms, and failure investigation in GPU-accelerated projects.

---

## Pipeline Architecture

### Multi-Stage Sequential Pipeline
<!-- source: SGLang -->

A well-designed CI pipeline uses **staged execution** with increasing resource requirements:

```
build / lint
  └─ check-changes ──── detect which packages changed
       │
       ├─ Stage A (~3 min) ── pre-flight: lightweight GPU + CPU tests
       │
       ├─ Stage B (~30 min) ── basic tests: 1-GPU, 2-GPU on various hardware
       │
       └─ Stage C (~30 min) ── advanced tests: 4-GPU, 8-GPU, multi-node
            │
            └─ finish job ── aggregates all results
```

**Key design principles:**
- Earlier stages use cheaper resources (CPU, small GPUs)
- Later stages use expensive resources (multi-GPU, specialized hardware)
- Each stage gates the next: Stage A failure prevents Stage B from consuming GPU hours
- A `finish` job aggregates all results as the final pass/fail gate

### Parallel Pipeline for Scheduled Runs
<!-- source: SGLang -->

| Aspect | PR runs | Scheduled runs (cron) |
|--------|---------|----------------------|
| Stage ordering | Sequential (A -> B -> C) | All stages in parallel |
| Fast-fail | Enabled | Disabled (run everything) |
| continue-on-error | No (stop at first failure) | Yes (collect all failures) |
| Concurrency | Cancel-in-progress per branch | Queue, no cancel |

### Container Build as Pipeline Stage
<!-- source: Megatron-LM -->

```
pre-flight
  └─ configure (scope, container tag)
       ├─ linting
       ├─ container-build
       │    ├─ parse-unit-tests → run-unit-tests
       │    └─ parse-integration-tests → run-integration-tests
       └─ final-gate
```

Container images are tagged by PR number and commit SHA, pushed to registry (ECR, GCR), and reused across all test jobs.

---

## Fast-Fail Mechanisms (4 Layers)
<!-- source: SGLang -->

From fine-grained to coarse-grained:

| Layer | Mechanism | Granularity | Description |
|-------|-----------|-------------|-------------|
| **1. Test -> File** | `unittest -f` / `pytest -x` | One test fails -> file stops | Failfast flag on test runner |
| **2. File -> Suite** | Suite runner default mode | One file fails -> suite stops | Controlled by `--continue-on-error` flag |
| **3. Job -> Job (same stage)** | `check-stage-health` action | One job fails -> sibling jobs fast-fail | Queries API for failed jobs in current run |
| **4. Stage -> Stage** | `wait-for-stage` + `needs` | Stage A fails -> Stage B/C skip entirely | Lightweight polling jobs between stages |

### Layer 3: Cross-Job Health Check

After checkout in every test job, a health check action:
1. Queries the workflow run API for all jobs
2. Filters for **root cause failures only** (excludes cascade failures)
3. If root cause found -> `core.setFailed()` with job names
4. Subsequent steps auto-skip via default `if: success()`

**Cascade filtering**: A job that fast-fails due to health check is itself marked failed. The filter excludes these by checking if the failing step name contains "check-stage-health", showing only the original root cause.

### Layer 4: Stage Gating (Polling)

Lightweight `ubuntu-latest` jobs poll the GitHub Actions API:
1. List all jobs in current run
2. Match jobs by name prefix (handles matrix jobs like `stage-b-test-1-gpu (3)`)
3. If any matched job has `conclusion: failure` -> fail immediately
4. If all matched jobs completed and count matches `expected_count` -> success
5. Otherwise -> sleep and retry (timeout after 4-8 hours)

**Critical**: `expected_count` must match matrix size. Adding/removing matrix entries requires updating the wait job spec.

---

## Test Partitioning (LPT Heuristic)
<!-- source: SGLang -->

Large test suites are split across matrix jobs using **Longest Processing Time (LPT)** partitioning:

1. Sort tests by estimated time descending (filename as tie-breaker for determinism)
2. Greedily assign each test to the partition with smallest cumulative time
3. Result: roughly equal total time per partition

**Workflow usage:**

```yaml
strategy:
  matrix:
    partition: [0, 1, 2, 3, 4, 5, 6, 7]
steps:
  - run: python3 run_suite.py --suite <name> \
           --auto-partition-id ${{ matrix.partition }} \
           --auto-partition-size 8
```

**Typical partition counts by resource tier:**

| Resource Tier | Example Partitions | Runner Type |
|---------------|-------------------|-------------|
| 1-GPU small | 1-8 | Consumer/small GPU |
| 1-GPU large | 8-14 | Datacenter GPU (H100) |
| 2-GPU | 2-4 | Multi-GPU node |
| 4-GPU | 1-4 | Multi-GPU node |
| 8-GPU | 2-4 | Full node |

---

## Retry Classification
<!-- source: SGLang -->

When a test fails, classify whether it should be retried:

**Non-retriable** (real code errors -- checked first):
`SyntaxError`, `ImportError`, `ModuleNotFoundError`, `NameError`, `TypeError`, `AttributeError`, `RuntimeError`, `CUDA out of memory`, `OOM`, `Segmentation fault`, `core dumped`, `ConnectionRefusedError`, `FileNotFoundError`

**Retriable** (flaky/accuracy):
`AssertionError` with comparison patterns (`not greater than`, `not less than`), `accuracy`, `score`, `latency`, `throughput`, `timeout`

**Default**: Unknown `AssertionError` -> retriable. Other unknown failures -> not retriable.

### Auto-Retry for Transient Infrastructure Failures
<!-- source: Megatron-LM -->

Some projects retry up to 3 times for known transient patterns:
- NCCL timeout
- ECC error
- Segfault (hardware-induced)
- HuggingFace connectivity failure

If all retries exhausted with transient pattern, it is infrastructure noise, not a code regression.

---

## Change Detection
<!-- source: SGLang -->

Determine which test suites to run based on file changes:

| Trigger | Detection Method |
|---------|-----------------|
| Pull request | GitHub diff (e.g., `dorny/paths-filter`) |
| Manual dispatch | GitHub API compare: `repos/{repo}/compare/main...{sha}` |
| Scheduled / force-all | Skip detection, run everything |

Output flags control which downstream stages/workflows execute (e.g., `main_package`, `kernel`, `multimodal`).

---

## Bot Commands / Slash Commands
<!-- source: SGLang -->

PR comment commands handled by a slash command handler workflow:

| Command | Effect |
|---------|--------|
| `/tag-run-ci-label` | Adds CI trigger label to PR |
| `/rerun-failed-ci` | Reruns failed jobs in latest workflow run |
| `/tag-and-rerun-ci` | Adds label + reruns |
| `/rerun-stage <stage>` | Dispatches workflow targeting a single stage |
| `/rerun-test <test-file>` | Reruns a specific test file |

<!-- source: FlashInfer -->

Some projects use bot commands on PRs to trigger specific CI actions:

| Command | Effect |
|---------|--------|
| `/bot run benchmark` | Trigger benchmark suite |
| `/bot run tests` | Trigger full test suite |

---

## CI Scope Labels
<!-- source: Megatron-LM -->

Control CI scope via PR labels:

| PR Label | Scope | Behavior |
|----------|-------|----------|
| _(none)_ | Slim | Lightweight subset, fast feedback |
| `Run tests` | Full | Full suite, lightweight mode |
| `Run functional tests` | Full + golden compare | 100-step training + golden value comparison |
| `container::lts` | LTS image | Use LTS base image instead of dev |

---

## CI Failure Investigation

### Step 1: Locate the Failure
<!-- source: Megatron-LM, SGLang -->

```bash
# List recent workflow runs for a PR
gh run list --branch "<branch-name>"

# Show summary of a specific run
gh run view <run-id>

# Stream failed job logs
gh run view <run-id> --log-failed
```

### Step 2: Download Full Logs

Runner stdout only shows partial output (e.g., ranks 0 and 3 out of 8). Full per-rank logs are uploaded as GitHub artifacts:

```bash
# List artifacts for a run
gh run view <run-id> --json artifacts --jq '.artifacts[].name'

# Download artifact
gh run download <run-id> --name "<artifact-name>" -D ./ci-logs

# Find files with errors
grep -r -l "ERROR\|Traceback\|FAILED\|fatal" ./ci-logs/

# Read logs in chunks (they can exceed 10,000 lines)
wc -l ./ci-logs/<path>/stderr.log
sed -n '1,200p' ./ci-logs/<path>/stderr.log
```

### Step 3: Classify the Failure

| Symptom | Likely Cause | What to Check |
|---------|-------------|---------------|
| Linting failure | Code style violations | Run autoformat locally |
| Container build failure | Dependency conflict or broken git-sourced package | Check container build job log, `uv.lock` / `requirements.txt` |
| Unit test failure in specific bucket | Code regression | Re-run bucket locally inside container |
| Functional test golden mismatch | Numerical regression | Compare loss curves, check if computation changed |
| All downstream jobs skipped | Earlier stage failed, fast-fail triggered | Find the root cause job (red X, not cascade) |
| `wait-for-stage` timeout | `expected_count` mismatch with matrix size | Verify job spec counts |
| Flaky test retried and passed | Accuracy/performance variance | Check `[CI Retry]` markers in logs |
| Tests pass locally but fail in CI | Partition assignment, GPU type, or env difference | Check which partition the test lands in; verify runner label |

### Step 4: Correlate with PR Changes

```bash
# Find tests that cover a changed source file
grep -r "from <module>.<changed_path>" tests/ -l

# Check if failure exists on main (pre-existing issue)
gh run list --branch main --limit 5
```

---

## Concurrency Control
<!-- source: SGLang -->

CI concurrency keys should include:
- Event name (prevents scheduled runs colliding with PR runs)
- Branch name (per-branch isolation)
- PR SHA (isolates re-run dispatches from main runs)
- Target stage (allows parallel stage dispatches)

```yaml
concurrency:
  group: pr-test-${{ github.event_name }}-${{ github.head_ref }}-${{ inputs.target_stage || 'all' }}
  cancel-in-progress: true  # New push cancels old run for PRs
```

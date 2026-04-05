---
name: bisect
description: Git bisect to find the commit that introduced a regression.
allowed-tools: [Bash, Read, Grep]
---

<!-- source: torchtitan -->

# Bisect

Use `git bisect` to find the commit that introduced a regression. Judge candidate commits
by command exit code, not by manually interpreting logs.

## Usage

```text
/bisect [--good <ref>] [--bad <ref>] [--build <cmd>] --test <cmd>
```

## Phase 1: Gather Information

Collect:

- target repo path
- known good ref or date
- known bad ref, usually `HEAD` or the current branch
- optional build command
- required test command
- timeout policy for long-running tests

Rules:

- use absolute paths when switching repos
- the test command must return `0` for good and non-zero for bad
- if the regression is flaky, do not start bisect until the signal is stable enough

## Phase 2: Setup

1. Verify the repo is clean enough for bisect:

```bash
git status --short
git bisect log
```

2. Fetch if needed:

```bash
git fetch origin
```

3. Resolve the good and bad refs.
4. Estimate search space:

```bash
git rev-list --count <good>..<bad>
```

5. Start bisect:

```bash
git bisect start
git bisect bad <bad>
git bisect good <good>
```

If any setup step fails, reset bisect before exiting.

## Phase 3: Bisect Loop

For each candidate commit:

### A. Build

If a build command is required, run it first. On build failure:

- show the failure
- let the user decide whether to mark it bad, skip it, or abort

### B. Test

Run the test command and use exit code only:

- exit `0` -> `git bisect good`
- non-zero -> `git bisect bad`
- timeout -> follow the chosen timeout policy

### C. Record Progress

After each step, report:

- current commit
- good/bad/skip result
- approximate remaining steps

Continue until Git reports the first bad commit or the search can no longer narrow.

## Phase 4: Report and Cleanup

Capture:

```bash
git bisect log
git show --stat <first_bad_commit>
```

Then reset:

```bash
git bisect reset
```

Final report should include:

- first bad commit hash
- commit subject, author, and date
- changed files summary
- full bisect log
- optional linked PR if the commit subject or remote metadata reveals one

## Rules

- Judge the test only by exit code.
- Keep the user informed after every bisect step.
- Reset bisect before exiting on unexpected errors.

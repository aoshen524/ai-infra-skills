<!-- source: AReaL -->

# PR Workflow

This guide combines Conventional Commits, PR review flow, and PR creation into one
practical workflow for AI infrastructure repositories.

## 1. Conventional Commits

### Format

```text
<type>(<scope>): <subject>
```

Scope is optional. Subject should be imperative, concise, and without a trailing period.

### Common Types

| Type | Use When |
| ---- | -------- |
| `feat` | adding a user-visible feature or capability |
| `fix` | correcting a bug or regression |
| `docs` | documentation-only change |
| `refactor` | code restructuring without behavior change |
| `test` | adding or fixing tests |
| `chore` | build, tooling, dependency, or maintenance work |
| `perf` | measurable performance improvement |

### Scope Rules

Infer scope from the dominant area touched:

- `workflow`
- `engine`
- `model`
- `kernel`
- `infra`
- `api`
- `docs`
- `tests`

If the change spans many unrelated areas, omit the scope rather than forcing a bad one.

### Examples

```text
fix(engine): handle sharded checkpoint resume
feat(workflow): add multi-turn tool trajectory support
docs: clarify launcher timeout troubleshooting
```

## 2. PR Review Workflow

Use a four-phase review loop.

### Phase 1: Analyze

Collect:

- PR title and description
- files changed and change size
- affected domains such as distributed runtime, numerics, model code, infra, or docs
- test coverage claims

Prefer commands such as:

```bash
gh pr view <pr_number> --json title,body,files,commits
git diff --stat <base>...<head>
```

### Phase 2: Plan Tasks

Split review work by risk area, not just by file count.

Good review buckets:

- distributed coordination and synchronization
- tensor shape, dtype, and masking semantics
- checkpointing and state compatibility
- infra and launch behavior
- documentation and tests

For each bucket, define:

- what could regress
- which files need to be read together
- what proof would confirm or reject the risk

### Phase 3: Execute

Read the changed code and enough surrounding context to answer:

- does the code do what the PR claims?
- are edge cases or fallback paths broken?
- do tests cover the risky paths?
- is there a likely silent numerical regression?

Prefer findings with file and line evidence.

### Phase 4: Score and Summarize

Score each finding by:

- severity
- confidence
- blast radius
- likelihood in production or CI

Report findings first, then open questions, then a short summary.

## 3. Create PR Workflow

### Step 1: Verify Local State

```bash
git branch --show-current
git status --short
gh --version
```

Do not create a PR from `main` or with unrelated uncommitted changes.

### Step 2: Rebase

```bash
git fetch origin <base_branch>
git rebase origin/<base_branch>
```

If conflicts occur, stop and resolve them before continuing.

### Step 3: Squash and Generate Commit Message

```bash
git diff --cached --name-only
git diff --cached
```

Choose commit type, infer scope, and generate a Conventional Commit subject.

### Step 4: Push and Create PR

```bash
git push -u <remote> <branch>
gh pr create --base <base_branch> --title "<title>" --body-file <body_file>
```

If the repo uses a fork workflow, detect a writable remote first and create the PR from
`<fork_owner>:<branch>` back to the upstream repo.

## 4. PR Description Template

```markdown
## Description

Briefly explain what changed and why.

## Related Issue

Fixes #<issue>

## Type of Change

- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update
- [ ] Refactor
- [ ] Performance improvement
- [ ] Test coverage improvement

## Validation

- [ ] Formatting passed
- [ ] Relevant tests passed
- [ ] Docs updated if needed
- [ ] Backward compatibility considered

## Additional Context

Anything reviewers should know: rollout plan, benchmarks, screenshots, logs, or follow-up work.
```

## Review Heuristics for AI Infra

- Distributed changes need both correctness review and topology review.
- Numerical changes need at least one parity signal, not just a green test.
- Config changes should be reviewed as API changes.
- Missing tests are more serious when the changed path has many parallel combinations.

---
name: create-pr
description: Rebase, squash, and create a PR on GitHub with intelligent title and description.
allowed-tools: [Bash, Read, Grep]
---

<!-- source: AReaL -->

# Create PR

Rebase onto the target base branch, squash local commits into a clean history, and
create or update a GitHub pull request with a Conventional Commit title and a structured
body.

## Usage

```text
/create-pr [--draft] [--base <branch>]
```

## Workflow

### 1. Verify Prerequisites

```bash
git branch --show-current
git status --short
gh --version
```

Rules:

- do not create a PR from `main` or `master`
- stop if there are unrelated uncommitted changes
- confirm the target base branch, defaulting to `main`

### 2. Determine Push Remote

Support both maintainer and fork workflows.

Checks:

```bash
git remote -v
git push --dry-run origin HEAD
```

If `origin` is not writable:

- find a fork remote
- determine the fork owner
- plan to open the PR from `<fork_owner>:<branch>` back to the upstream repo

### 3. Check for an Existing PR

```bash
gh pr view --json number,title,url 2>/dev/null
```

If a PR already exists for the branch, warn the user that rebasing and squashing may
rewrite history before continuing.

### 4. Fetch and Rebase

```bash
git fetch origin <base_branch>
git rebase origin/<base_branch>
```

If rebase conflicts occur, stop and let the user resolve them manually.

### 5. Squash Commits

Count commits since the base branch, then squash into one clean commit:

```bash
git rev-list --count origin/<base_branch>..HEAD
git reset --soft origin/<base_branch>
```

Generate a Conventional Commit title from the staged diff before committing.

### 6. Generate PR Title and Description

PR title format:

```text
<type>(<scope>): <subject>
```

PR body template:

```markdown
## Description

What changed and why.

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
- [ ] Documentation updated if needed
- [ ] Backward compatibility considered

## Additional Context

Benchmarks, rollout notes, screenshots, logs, or follow-up work.
```

Fill the template from the actual diff, not from generic prose.

### 7. Confirm, Push, and Create or Update the PR

Show the user:

- chosen remote
- base branch
- final PR title
- final PR body
- whether the PR will be draft or ready

Then run:

```bash
git push -f -u <push_remote> <branch>
gh pr create --base <base_branch> --title "<title>" --body "<body>"
```

If the PR already exists, use `gh pr edit` instead of `gh pr create`.

## Fork Workflow Notes

- push to the writable fork remote
- target the upstream repository with `--repo`
- set the PR head to `<fork_owner>:<branch>`

## Guardrails

- Never hide a rebase conflict.
- Never force-push without telling the user it will rewrite branch history.
- Keep the PR focused on one concern after squashing.

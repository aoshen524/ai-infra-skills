# Knowledge Ingest

<!-- source: Awesome-ML-SYS-Tutorial /learn proposal -->

Use this guide when the task is to absorb valuable external material into
`ai-infra-skills` without turning the repo into a loose pile of copied notes.

This workflow is for articles, design docs, tutorials, PRs, or external repos that carry
reusable patterns.

## When To Use

Use it for:

- reading an article and deciding what should enter this repo
- turning a design doc or tutorial into skill or knowledge updates
- converting one-off research into durable repo assets
- deciding whether a finding belongs in `knowledge`, `skills`, `agents`, `commands`, or only the audit layer

Do not use it for:

- blind summarization with no repo change
- routine one-line doc edits
- upstream repo audits that already fit `.claude/skills/04-review-plan/upstream-refresh.md`

## Core Ideas

This workflow borrows four durable ideas from structured learning systems:

1. **Concept -> model -> code**
   - understand the conceptual frame first, then the mechanism, then the implementation surface
2. **Progressive derivation**
   - preserve why a pattern exists, not just what command or API was used
3. **Knowledge graph discipline**
   - every absorbed insight should have a clear home, owner, and adjacency
4. **Staged authoring**
   - plan, write, review, then register in the repo indexes

## Ingest Contract

Before changing files, lock:

- source material
  - exact article, PR, repo, or document
- extraction scope
  - which ideas are under consideration
- target audience
  - agent maintainer, serving engineer, kernel engineer, RL workflow engineer, and so on
- expected repo surfaces
  - `knowledge`, `skills`, `agents`, `commands`, indexes, or audit docs

If the source is broad, narrow the scope to the specific reusable patterns first.

## Decision Rules

### Absorb

Absorb a finding when it is:

- durable across tasks
- reusable by agents or maintainers
- better as a repo asset than as a temporary note
- grounded in a mechanism rather than only a result

### Watch

Mark it as watch-only when:

- it is promising but still immature
- it depends on unstable upstream behavior
- it needs more examples before becoming canonical guidance

### Ignore

Ignore it when:

- it is only a narrative flourish
- it duplicates stronger guidance we already have
- it has no clear landing surface in this repo

## Landing Rules

Route the extracted material to the correct layer:

- `knowledge/*` for stable mechanisms, architecture constraints, and design context
- `.claude/skills/*` for executable workflows and repeatable maintenance routines
- `.claude/agents/*` when expert boundaries or owned knowledge changed
- `.claude/commands/*` only when the pattern is a repeatable, compact command workflow
- `INDEX.md`, `KNOWLEDGE_INDEX.md`, `EXPERT_KNOWLEDGE_MAP.md`, `CLAUDE.md`, `README.md`, `COVERAGE_AUDIT.md` when discoverability changes

Prefer a skill when the value is procedural.

Prefer a knowledge doc when the value is conceptual or architectural.

Prefer both only when the pattern has a stable mechanism plus an execution workflow.

## Recommended Workflow

### Phase 1: Read And Extract

Capture:

- the motivating problem
- the core mechanism
- the constraints or tradeoffs
- the reusable engineering pattern
- the likely audience in this repo

Do not start from prose style or anecdote. Start from the mechanism.

### Phase 2: Decide The Repo Shape

Ask:

1. is this a stable concept or a repeatable workflow?
1. which expert owns the closest topic?
1. do we already have a nearby doc that should be extended instead of creating a new file?
1. does the finding change navigation or ownership?

### Phase 3: Write The Smallest Durable Asset

Create or update only the minimum needed:

- extend an existing canonical doc if possible
- add one new knowledge doc if the concept deserves a stable home
- add one thin skill if the execution workflow needs to be routable

Do not create multiple sibling docs unless the topic truly splits into separate concepts.

### Phase 4: Register The Knowledge

If the ingest created or materially changed repo structure, update:

- `INDEX.md`
- `KNOWLEDGE_INDEX.md`
- `EXPERT_KNOWLEDGE_MAP.md`
- `CLAUDE.md`
- `README.md` or `COVERAGE_AUDIT.md` when source attribution or audit state changed

### Phase 5: Review The Ingest

Verify:

- the new material is easier to find than before
- the owner expert is explicit
- the mechanism is preserved
- the repo did not gain duplicated or overlapping leaf docs

## Preferred Output Shape

```markdown
## Source

## Reusable Findings

## Repo Landing Decisions

## Changes Made

## Follow-up Watch Items
```

## Review Questions

1. does the absorbed content preserve the mechanism, not just the conclusion?
1. is the landing layer correct for the type of insight?
1. did we update indexes when the repo surface changed?
1. will another agent find and use this pattern later without rediscovering it?

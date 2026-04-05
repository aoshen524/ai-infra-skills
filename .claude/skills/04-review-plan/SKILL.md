---
name: review-plan
description: Plan review skill for complex implementation tasks. Creates structured plans with domain classification, risk assessment, and phased execution. Use before multi-file changes, new features, or architectural decisions.
model: claude-opus-4-6
tools: [Read, Grep, Glob, Bash]
---

# Plan Review

<!-- source: AReaL, TensorRT-LLM -->

This category is for planning and review structure, not domain knowledge itself.

Use it to:

- turn a multi-file task into a compact execution plan
- classify change risk by domain
- review completed work against explicit checklists

## When to Activate

Use this category when:

- the task spans multiple files or modules
- a new workflow, engine path, or API surface is being added
- you need a risk-aware implementation plan before coding
- you need strict post-change review against a checklist

Do not use it for:

- obvious single-file edits
- typo or prose-only changes
- deep domain reasoning that should go to an expert first

## Included Guides

| Guide | Use For |
| ----- | ------- |
| [review-domains.md](review-domains.md) | detect domains, signals, and review depth |
| [review-templates.md](review-templates.md) | run domain-specific checklist review |

## Planning Contract

Every useful plan should answer only the essentials:

1. what is changing
1. which files or modules are affected
1. what existing pattern should be copied
1. what the main risk is
1. how the result will be validated

Use a quick path for small multi-file changes and a fuller plan only when there are real
risks or multiple phases.

## Review Contract

When this category is used for review:

1. do not trust summary claims without reading files
1. cite concrete evidence for pass and fail decisions
1. mark ambiguity as fail or not-yet-proven
1. keep output checklist-shaped instead of essay-shaped

## Minimal Output Shapes

### Plan

```markdown
## Summary

## Changes

## Steps

## Risks

## Testing
```

### Review

```text
REVIEW RESULT: PASS | FAIL

=== Section ===
item  PASS|FAIL  file:line  evidence
```

## Routing Rules

- if the task is domain-heavy, consult the relevant expert before finalizing the plan
- use `review-domains.md` to detect the right risk class
- use `review-templates.md` to avoid inventing checklists from scratch

This category should stay short. The detailed taxonomy and checklists live in the two
child guides.

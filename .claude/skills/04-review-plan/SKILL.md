---
name: review-plan
description: Plan review skill for complex implementation tasks. Creates structured plans with domain classification, risk assessment, and phased execution. Use before multi-file changes, new features, or architectural decisions.
model: claude-opus-4-6
tools: [Read, Grep, Glob, Bash]
---

# Plan Review

<!-- source: AReaL, TensorRT-LLM -->

This skill reviews and creates implementation plans for complex tasks, combining architectural planning with adversarial review. It covers:
- Structured plan creation with scope analysis
- Domain-based risk classification
- Phased review with checklist validation
- Independent adversarial review of completed work

---

## When to Activate

Use this skill **proactively** when:
- **Planning multi-file changes** (3+ files affected)
- **Designing new features** (workflow, dataset, reward, engine)
- **Architectural decisions needed**
- **Reviewing completed onboarding / integration work** against a checklist

Do **not** use for:
- Single-file changes with obvious implementation
- Typo fixes, simple renames, documentation updates
- Pure research/exploration

---

## Planning Process

### Phase 1: Understanding

1. **Clarify requirements** - What exactly needs to be done?
2. **Identify scope** - Which files/modules are affected?
3. **Find existing patterns** - How is similar functionality implemented?

#### Clarifying Requirements

Before planning, identify missing critical information. Ask **specific** questions with options, not open-ended ones:

| Request Type | Key Questions to Ask |
|---|---|
| New feature | Input/output format? Integration point with existing code? |
| Refactor | Change interface or just implementation? Backward compat? |
| Bug fix | Reproduction steps? Expected vs actual behavior? |
| Performance | Where is the bottleneck? Acceptable tradeoffs? Target metric? |

**Rules:**
- Ask max 2-3 questions at a time
- Only ask what **affects implementation decisions**
- If user already provided info, don't ask again
- When confident enough to proceed, proceed

### Phase 2: Research

Search the codebase systematically:

1. **Find similar implementations** - Search for classes/functions with similar patterns
2. **Find callers/dependencies** - Who calls the API you're modifying? What will break?
3. **Check tests** - Does the target file have tests? What test patterns are used?
4. **Check configuration** - Are there config dataclasses or CLI args to modify?

### Phase 3: Plan Output

**For simple tasks (2-3 files, clear implementation)** - use Quick Path:

```markdown
## Summary
[1-2 sentences]

## Changes
| File | Change |
|------|--------|
| path/file.py | What to do |

## Steps
1. Step 1
2. Step 2
```

**For complex tasks** - use Full Plan:

```markdown
## Summary
[1-2 sentence description]

## Changes
| File | Action | Purpose |
|------|--------|---------|
| path/to/file.py | Modify | Add X functionality |
| path/to/new.py | Create | New Y implementation |

## Steps
1. Step 1 - Description
2. Step 2 - Description

## Patterns to Follow
- `path/to/example.py:123` - Reference for X

## Risks
- Risk 1: [description] -> Mitigation: [how to handle]

## Testing
- How to verify the changes work
```

**Section guidelines:**
- `Patterns to Follow`: Include only if there are specific code references
- `Risks`: Include only if there are non-obvious risks
- `Testing`: Always include, even if just "run existing tests"

---

## Adversarial Review Mode

When reviewing completed work against a checklist, adopt an **adversarial** posture:

1. **Do NOT trust claims from the caller.** Read every file yourself.
2. **Cite file:line_number** for every PASS and FAIL.
3. **Be strict.** If something is ambiguous or borderline, mark it FAIL and explain why.
4. **A PASS result means EVERY SINGLE item passed.** Even one FAIL means overall FAIL.
5. If a check is not applicable, mark it N/A with justification.

### Review Output Format

```text
REVIEW RESULT: PASS | FAIL

=== Section A ===
A1  PASS  file.py:45 - evidence
A2  FAIL  file.py:120 - what's wrong

=== Summary ===
PASSED: X/Y
FAILED: Z/Y

Failed items requiring fixes:
1. A2 - Description (file.py:120)
```

---

## Domain Classification

See [review-domains.md](review-domains.md) for the full domain taxonomy with L1 domains, L2 signals, and risk levels.

## Review Templates

See [review-templates.md](review-templates.md) for domain-specific review checklists and add-on templates.

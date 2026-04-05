---
name: code-review
description: PR workflow, commit conventions, and CI failure triage for AI infrastructure projects
model: claude-opus-4-6
tools: [Bash, Read, Grep, Glob]
---

# Code Review

<!-- source: AReaL, TensorRT-LLM, SGLang, torchtitan -->

This skill covers the practical review workflow around commits, PRs, and CI failures in
AI infrastructure projects.

## Included Guides

| Guide | When to Use |
| ----- | ----------- |
| [pr-workflow.md](pr-workflow.md) | Reviewing a PR, creating a PR, or generating Conventional Commit messages |
| [ci-failure-triage.md](ci-failure-triage.md) | Investigating CI failures, regressions, flaky tests, or pipeline breakages |

## When to Activate

Use this skill when:

- preparing a commit, PR, or review summary
- evaluating a complex PR that spans multiple domains
- triaging CI failures before or after review
- deciding whether a regression is code, infra, environment, or flakiness

## Cross-Domain Principles

Distributed systems are rarely fully testable through exhaustive local runs. Review with
that constraint in mind:

1. **Enumerate parallel combinations**. Check the changed code against the relevant
   combinations of TP, DP, PP, CP, EP, batch size, and world size instead of assuming one
   tested path proves correctness.
1. **Validate numerical behavior**. In AI infra, "it runs" is not enough. Check loss,
   logits, precision, mask semantics, and checkpoint parity when numerics are touched.
1. **Check fallback paths**. Fast paths often get tested first, but CPU fallback,
   unsharded mode, and single-device mode are common escape hatches and debugging tools.
1. **Review config contracts**. Many regressions come from schema drift or missing config
   propagation rather than obvious code bugs.
1. **Prefer evidence over intuition**. Use logs, diffs, repro commands, and CI artifacts
   to support findings.

## Typical Workflow

1. Use [pr-workflow.md](pr-workflow.md) to assess change intent, create a commit or PR,
   and produce a structured review.
1. Use [ci-failure-triage.md](ci-failure-triage.md) when local validation is not enough
   or CI signals disagree with local results.

<!-- source: TensorRT-LLM, SGLang, torchtitan -->

# CI Triage Patterns

## What This Covers

This document captures the repeatable patterns for turning a failing CI run into a
useful diagnosis.

Use it for:

- failure bucketing
- regression bisection
- runner or environment drift analysis
- separating infra noise from code regressions

## Triage Loop

1. capture the first specific error
1. record run ID, job ID, SHA, runner type, and failing workload
1. bucket the failure by subsystem
1. compare pass and fail environments
1. reproduce with the smallest useful command
1. bisect only after the signature is stable

## Buckets

Recommended buckets:

- `infra/resource`
- `infra/runtime`
- `env/dependency`
- `config/schema`
- `distributed/sync`
- `numerics/semantics`
- `model/artifact`
- `backend/compiler`

Avoid a catch-all bucket. If you cannot name the subsystem, investigation is not done.

## Boundary Rule

Always distinguish between:

- code regression
- environment drift
- hardware-specific behavior
- flakiness

The same symptom can come from all four, but the fix path is different.

## Bisection Rule

Only run `git bisect` when:

- pass and fail boundaries are credible
- runner differences have been checked
- the test command has a stable exit-code contract

## Review Questions

1. what is the earliest specific failure, not the wrapper error?
1. is this one bucket or multiple unrelated buckets?
1. did runner type, image, driver, or pool change?
1. can the failure be reproduced with a narrower command before deeper investigation?

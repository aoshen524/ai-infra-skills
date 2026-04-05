<!-- source: AReaL, veRL, SLIME -->

# RL Workflow Contracts

## What This Covers

This document captures the stable contracts behind RL rollout workflows, rollout
functions, and reward wiring.

Use it for:

- adding a new workflow
- designing a multi-turn agent loop
- wiring evaluation and rollout together
- validating what counts as a trainable trajectory

## Three-Layer Contract

Most RL systems expose three extension layers:

```text
workflow -> rollout function -> reward function
```

- the workflow defines one episode contract
- the rollout layer turns many episodes into train or eval batches
- the reward layer scores one sample or one group

Keep these layers separate. Hidden coupling across them makes debugging painful.

## Workflow Contract

A healthy workflow should make explicit:

- input sample schema
- generation path
- reward invocation
- output trajectory schema
- rejection behavior

For async systems, the workflow surface should stay async end to end.

## Agent Loop Contract

For multi-turn or tool-using RL, preserve token trajectories directly instead of trying
to reconstruct them from final chat history.

Minimum useful outputs:

- `prompt_ids`
- `response_ids`
- `response_mask`

This avoids policy-trajectory drift after tool parsing or decode-reencode steps.

## Rollout Function Contract

The rollout surface should clearly distinguish:

- train mode
- eval mode
- required sample fields
- batch grouping semantics

When the same rollout surface supports eval, dataset overrides and reward keys should be
explicit rather than implicit.

## Reward Contract

Reward functions should make clear:

- scalar vs grouped return shape
- error handling for invalid metadata
- whether post-processing exists
- which key downstream code should treat as the training reward

## Failure Modes

| Symptom | Likely cause | First check |
| ------- | ------------ | ----------- |
| no learning despite valid generations | reward key or mask mismatch | inspect output sample fields and reward wiring |
| eval works but train fails | rollout branches diverged | compare train and eval path assumptions |
| multi-turn instability | token trajectory drift | inspect explicit token outputs, not only chat messages |
| rejected samples silently disappear | rejection semantics unclear | check explicit `None` or status handling |

## Review Questions

1. are workflow, rollout, and reward layers clearly separated?
1. is the trainable trajectory schema explicit?
1. are multi-turn token trajectories preserved directly?
1. do train and eval paths share one clear contract?

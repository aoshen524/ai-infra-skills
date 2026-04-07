<!-- source: AReaL, veRL, SLIME, OpenRLHF, XTuner -->

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

## Agent Executor Contract

OpenRLHF highlights a useful additional separation:

```text
agent executor mode -> rollout workflow -> RL algorithm
```

Where:

- the executor mode decides whether interaction is single-turn or multi-turn
- the workflow manages trajectory production and reward wiring
- the RL algorithm consumes the resulting token trajectory without owning interaction semantics

This separation matters because:

- single-turn and multi-turn interaction modes should not require rewriting PPO-family logic
- reward computation and environment stepping should stay above the optimizer layer
- the trainable object is still a token trajectory even when the environment is richer

### Practical Rule

Keep execution mode orthogonal to algorithm choice whenever possible.

Good systems let:

- PPO
- REINFORCE-family methods
- GRPO-like methods

consume the same trajectory contract whether the interaction was one-shot or multi-step.

### OpenAI-Compatible Agent Bridge

An OpenAI-compatible chat surface can still be RL-safe, but only if the executor keeps
the trainable object as a token trajectory.

Prefer:

- protocol compatibility at the serving edge
- token IDs and logprobs captured during execution
- one explicit session or trajectory identity that survives tool-use round trips

Avoid:

- rebuilding training samples from final chat text only
- hiding prefix reuse or delta tokenization from trajectory accounting
- letting the gateway define the training schema implicitly

The main rule is simple:

- API compatibility is a serving concern; trajectory fidelity is an RL contract

## Rollout Function Contract

The rollout surface should clearly distinguish:

- train mode
- eval mode
- required sample fields
- batch grouping semantics

When the same rollout surface supports eval, dataset overrides and reward keys should be
explicit rather than implicit.

## Dataset Pipeline Contract

The rollout contract is only trustworthy if the dataset pipeline contract is also stable.

At minimum, the system should make explicit:

- what one sample looks like before batching
- which fields packing and sampling depend on
- how resume restores global progress

See `knowledge/rl/dataset-pipeline-contracts.md` for the data-side contract.

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

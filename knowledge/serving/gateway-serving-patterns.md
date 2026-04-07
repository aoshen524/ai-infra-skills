<!-- source: AReaL, vLLM, OpenRLHF, XTuner -->

# Gateway Serving Patterns

## What This Covers

This document captures the serving-side patterns for exposing a familiar inference API
while collecting RL-ready trajectory data.

Use it for:

- OpenAI-compatible gateways
- session-scoped serving state
- reward writeback paths
- online RL serving loops
- rollout-worker memory lifecycle when serving and training share GPU state

## Core Pattern

```text
client -> gateway -> session store -> inference backend -> reward writeback -> batch consumer
```

The serving layer should preserve client compatibility while the training layer handles
trajectory validation and batching.

## Required Pieces

- a familiar request surface such as `/chat/completions`
- session-scoped identity and API keys
- a reward write path such as `POST /rl/set_reward`
- a trajectory assembly layer separate from raw chat responses
- token-level trace collection bound to session or trajectory identity
- backpressure and concurrency controls

## OpenAI-Compatible Executor Bridge

One of the most reusable patterns from recent RL serving systems is:

```text
client -> OpenAI-compatible gateway -> model backend -> token trace sink -> RL batch assembler
```

This matters because many agent stacks want standard chat APIs, while RL training still
needs exact token trajectories.

Design rules:

- keep protocol compatibility at the gateway boundary
- collect token IDs and logprobs below that boundary, not from reconstructed chat text
- preserve one stable session or trajectory ID across multi-turn calls
- allow prefix reuse only when trajectory accounting remains exact

If a system only stores final messages, it is easy for the serving path and training path
to disagree silently.

## Worker Memory Lifecycle

Serving-coupled RL systems often need more control than simple "restart the worker."

Treat memory-state operations as explicit lifecycle verbs:

- `onload_weights`
- `onload_kvcache`
- `offload`
- `reset_prefix_cache`

This separation helps a controller decide whether to:

- reclaim full model memory
- preserve warm KV state
- clear stale prompt cache
- recover one worker without rebuilding the entire serving tier

Good design rules are:

- separate weight residency from KV-cache residency
- make prefix-cache resets explicit and auditable
- do not overload `restart` to mean every memory-state transition
- surface these operations at the controller layer, not only inside worker-local code

## Design Rules

- keep serving semantics separate from training semantics
- allow standalone inference outside active RL sessions for debugging
- log readiness transitions, not only generated responses
- make session ownership and cache ownership explicit
- keep gateway config separate from model-engine config

## Failure Modes

| Symptom | Likely cause | First check |
| ------- | ------------ | ----------- |
| chat works but no batch appears | reward never bound to an interaction | inspect reward auth and trajectory IDs |
| sessions leak into each other | cache ownership bug | inspect session resolution and session-scoped keys |
| rollout becomes stale | backpressure is missing | inspect concurrency limits and queue size |
| real client fails while local tests pass | protocol mismatch | inspect request and SSE compatibility |
| RL batches cannot be reconstructed exactly | token trace was inferred from text only | inspect token-level trace capture and session binding |
| worker recovery clears too much or too little state | memory lifecycle verbs are collapsed together | inspect separate handling for weights, KV cache, and prefix cache |

## Review Questions

1. can a client connect with only `base_url` and credentials changed?
1. is reward submission session-scoped and auditable?
1. does the system log when a trajectory becomes trainable?
1. are serving and RL semantics separated cleanly enough to evolve independently?
1. can the controller distinguish weight, KV-cache, and prefix-cache lifecycle operations?

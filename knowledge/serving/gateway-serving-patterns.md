<!-- source: AReaL, vLLM -->

# Gateway Serving Patterns

## What This Covers

This document captures the serving-side patterns for exposing a familiar inference API
while collecting RL-ready trajectory data.

Use it for:

- OpenAI-compatible gateways
- session-scoped serving state
- reward writeback paths
- online RL serving loops

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
- backpressure and concurrency controls

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

## Review Questions

1. can a client connect with only `base_url` and credentials changed?
1. is reward submission session-scoped and auditable?
1. does the system log when a trajectory becomes trainable?
1. are serving and RL semantics separated cleanly enough to evolve independently?

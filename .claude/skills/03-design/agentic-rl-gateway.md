# Agentic RL Gateway Pattern

Patterns for integrating an existing agent runtime into an RL training loop through a
proxy that speaks a familiar inference API.

<!-- source: AReaL (GatewayInferenceController, inference service, OpenAI proxy) -->

## When to Use This Pattern

Choose a gateway-based design when:

- the agent already runs outside the trainer and should stay that way
- you want human-in-the-loop or external-agent online RL without rewriting the agent
- the cheapest integration surface is an OpenAI-compatible chat API
- rollout data must flow back into RL training with session and reward bookkeeping

Do not use this pattern when the workflow is already a simple in-process rollout
function. In that case, a direct workflow integration is usually simpler.

## Core Idea

Instead of forcing every agent into trainer-specific code, expose a stable serving
surface and collect RL metadata around it:

```text
external agent or human client
    -> OpenAI-compatible gateway
    -> session store + routing layer
    -> inference backend
    -> reward submission API
    -> trajectory bundler
    -> trainer batch consumer
```

This preserves compatibility with existing SDKs while still producing RL-ready
trajectories.

## Required Components

### 1. Familiar request surface

Use a protocol the agent can already speak, such as `/chat/completions`. This removes
integration friction and lets external tools point at the gateway with only a `base_url`
and API key change.

### 2. Session-scoped identity

Each training session should get its own session ID and session-scoped API key.

Why it matters:

- requests must be attributable to the right rollout
- reward writes must not cross sessions
- active completions and conversation state need a stable home

### 3. Reward submission endpoint

Expose a separate reward write path such as:

```text
POST /rl/set_reward
```

The reward endpoint should:

- authenticate the session
- identify the interaction or trajectory being scored
- report whether the trajectory is now ready for batching

### 4. Trajectory assembly

Do not treat chat completion responses as the training sample itself. A separate layer
should assemble:

- prompt or conversation history
- model outputs
- per-turn metadata
- reward
- completion status
- optional tracing fields

### 5. Router and concurrency controls

The gateway should own routing and backpressure instead of pushing those concerns into
the agent.

Useful controls:

- routing strategy such as round robin or sticky routing
- max concurrent rollouts
- queue size
- request and startup timeouts
- off-policy or staleness limits

## Generic Configuration Shape

```python
@dataclass
class GatewayConfig:
    tokenizer_path: str = ""
    model_path: str = ""
    routing_strategy: str = "round_robin"
    request_timeout: float = 120.0
    setup_timeout: float = 300.0
    consumer_batch_size: int = 16
    max_concurrent_rollouts: int | None = None
    max_head_offpolicyness: int = 0
    queue_size: int | None = None
    enable_rollout_tracing: bool = False
    dump_to_file: bool = False
```

Design rules:

- keep gateway config separate from model-engine config
- model startup knobs and rollout batching knobs should not share one flat namespace
- expose timeouts explicitly; hidden defaults make online RL hard to debug

## Integration Contract for Agent Workflows

The agent-side contract should be minimal. A generic async shape is:

```python
async def run(data: dict[str, Any], **extra_kwargs: Any) -> dict[str, float] | float:
    ...
```

Useful injected kwargs:

- `base_url`
- `api_key`
- `http_client`

This keeps the training system responsible for lifecycle, while the agent remains a
normal API client.

## Operational Rules

### Default to compatibility, then add RL behavior

Requests on the familiar inference endpoint should keep working even when they are not
part of an active RL session. This makes debugging far easier because the same gateway
can support:

- standalone inference
- scripted rollouts
- online RL sessions

### Log readiness transitions

The important event is not "a response was generated". It is "a trajectory became ready
for batching". Emit logs for:

- session creation
- gateway ready
- reward accepted
- trajectory ready
- rollout batch completed

### Keep reward writes idempotent where possible

Human feedback and external automation are noisy. Double submissions happen. Prefer
interaction IDs or trajectory IDs that let the service reject or safely overwrite
duplicates.

### Preserve chat semantics and training semantics separately

Streaming, retry handling, and tool traces belong to the serving layer. Reward shaping,
trajectory validation, and off-policy filtering belong to the training layer. Mixing the
two makes the system hard to evolve.

## Failure Modes to Check First

| Symptom | Likely cause | First check |
| ------- | ------------ | ----------- |
| client can chat but no training batch appears | rewards never attached to interactions | verify reward endpoint auth and interaction IDs |
| rewards arrive but rollout never completes | batch consumer or readiness transition is blocked | inspect queue size, consumer logs, and batch thresholds |
| some sessions receive another user's reward | session scoping bug | audit session key resolution and cache ownership |
| online RL feels stale | rollout queue outruns trainer | add concurrency limits and staleness caps |
| gateway works locally but not for real clients | protocol mismatch | confirm OpenAI-compatible request and SSE response format |

## Review Checklist

Before approving a gateway-based RL integration, confirm:

1. external agents only need `base_url` and credentials to connect
1. reward submission is authenticated and tied to an interaction identifier
1. session state and active completions are isolated per user or rollout
1. batching and backpressure are explicit in config
1. logs make it obvious when a trajectory becomes trainable
1. standalone inference still works for debugging

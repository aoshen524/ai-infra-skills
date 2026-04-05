# AI Infra Skills Coverage Audit

Last audited: 2026-04-05

This document records how the 12 source repositories are represented inside
`ai-infra-skills`, where their highest-value patterns landed, and which areas are
future-expansion candidates rather than current blockers.

## Audit Result

The repository now covers all 12 source repos across:

- `rules` for durable constraints
- `skills` for repeatable workflows
- `agents` for expert guidance
- `commands` for review/commit/bisect flows
- `knowledge` for deep technical references

The repository also now has an explicit expert-owned knowledge layer via
`EXPERT_KNOWLEDGE_MAP.md`, so domain experts, knowledge references, and workflow skills
can be navigated in a stable order.

Hardware coverage was also expanded into an expert-owned layer instead of staying as
ad hoc benchmark context:

- `knowledge/hardware/gpu-spec-reference.md` defines the hardware-note template
- `knowledge/hardware/nvidia-datacenter-gpu-matrix.md` anchors NVIDIA generation choices
- `knowledge/hardware/b300-blackwell-notes.md` keeps new-hardware bring-up guidance explicit

Two hard gaps found during audit were closed:

1. `02-env-source-log/debug-cuda-crash.md`
2. `04-review-plan/review-templates.md`

Underrepresented repos were also strengthened during this audit:

- `AReaL` via `moe-engine-expert.md`
- `AReaL` via `03-design/agentic-rl-gateway.md`
- `AReaL` via `knowledge/rl/tree-training.md`
- `AReaL` via `knowledge/distributed/fsdp-engine-patterns.md`
- `AReaL` via `knowledge/distributed/moe-engine-patterns.md`
- `AReaL` via `knowledge/distributed/launcher-scheduler-patterns.md`
- `SLIME` and `veRL` via `03-design/add-workflow.md`
- `vLLM` via `01-server/SKILL.md`
- `vLLM` via `knowledge/serving/disaggregated-serving.md`
- `Ollama` via `03-design/model-onboard.md`
- `Megatron-LM` via `rules/api-config.md`
- `Megatron-LM` via `knowledge/distributed/megatron-engine-patterns.md`
- `SGLang` via `rules/testing.md`
- `SGLang` via `knowledge/ci-cd/ci-triage-patterns.md`

Recent follow-up research also strengthened SGLang-derived workflow coverage in four
specific places:

- remote GPU backend execution as a first-class `01-server` workflow
- CUDA crash logging with offline replay artifacts and graph-safe behavior
- auto-driven benchmark search with resume-friendly outputs
- torch profiler triage tied back to Python scopes and known fusion patterns

## Expertization Status

| Theme | Primary expert(s) | Knowledge ownership | Status |
| ----- | ----------------- | ------------------- | ------ |
| RL algorithms | `algorithm-expert` | expert-owned | complete |
| RL workflow contracts | `rl-workflow-expert` | expert-owned | complete |
| FSDP engines | `fsdp-engine-expert` | expert-owned | complete |
| Megatron engines | `megatron-engine-expert` | expert-owned | complete |
| MoE engines | `moe-engine-expert` | expert-owned | complete |
| Distributed runtime and NCCL | `distributed-systems-expert` | expert-owned | complete |
| Kernel debugging and performance | `kernel-performance-expert` | expert-owned | complete |
| Hardware references | `distributed-systems-expert`, `kernel-performance-expert` | expert-owned | complete |
| Serving systems | `serving-systems-expert` | expert-owned | complete |
| Launcher and scheduler | `launcher-scheduler` | expert-owned | complete |
| Model onboarding debug | `model-debug-agent` | expert-owned | complete |
| CI triage | `ci-triage-expert` | expert-owned | complete |
| Post-change verification | `code-verifier` | utility-only | intentional |

## Coverage Matrix

| Repo | Core value extracted | Current placement | Status |
| ---- | -------------------- | ----------------- | ------ |
| `AReaL` | review flow, agents, FSDP/Megatron/MoE engine usage, RL algorithm patterns, launcher/scheduler, distributed debug, gateway-based online RL, tree training | `.claude/rules/*`, `.claude/agents/*`, `.claude/skills/03-design/*`, `.claude/skills/04-review-plan/*`, `.claude/skills/06-code-review/*`, `knowledge/distributed/debug-distributed.md`, `knowledge/rl/tree-training.md` | covered |
| `Megatron-LM` | container-first dev, uv/build/test flow, API backward compatibility, PR discipline | `.claude/skills/02-env-source-log/build-and-test.md`, `.claude/skills/02-env-source-log/ci-workflow.md`, `.claude/rules/api-config.md`, `.claude/agents/megatron-engine-expert.md` | covered |
| `TensorRT-LLM` | model onboarding, serve config reasoning, CI failure triage, pipeline/model debug | `.claude/skills/03-design/model-onboard.md`, `.claude/agents/model-debug-agent.md`, `.claude/skills/06-code-review/ci-failure-triage.md` | covered |
| `flash-attention` | kernel failure analysis, racecheck/TMA hazards, SASS/MMA, tuning | `knowledge/kernels/*`, `.claude/skills/05-profiler/*`, `.claude/skills/03-design/add-kernel.md` | covered |
| `flashinfer` | JIT architecture, kernel benchmark patterns, crash logging | `knowledge/kernels/jit-architecture.md`, `.claude/skills/05-profiler/benchmark-kernel.md`, `.claude/skills/02-env-source-log/debug-cuda-crash.md` | covered |
| `nccl` | low-level communication development patterns | `knowledge/distributed/nccl-patterns.md`, `.claude/rules/distributed.md` | covered |
| `ollama` | model packaging manifest, adapter wiring, runtime troubleshooting | `.claude/skills/03-design/model-onboard.md`, `.claude/skills/01-server/SKILL.md` | covered |
| `sglang` | CI architecture, remote GPU backend workflows, auto-benchmark, profiling, crash logging, test-writing conventions, CI bisect, torch.compile coverage checks | `.claude/skills/01-server/*`, `.claude/skills/02-env-source-log/*`, `.claude/skills/05-profiler/*`, `.claude/skills/06-code-review/ci-failure-triage.md`, `.claude/rules/testing.md`, `knowledge/ci-cd/sglang-ci-system.md`, `knowledge/serving/torch-compile-coverage.md` | covered |
| `slime` | reward/rollout extension points, eval dataset config, tests/CI integration | `.claude/skills/03-design/add-workflow.md`, `.claude/rules/testing.md` | covered |
| `torchtitan` | config safety, model structure, numerics validation, experiments isolation, bisect workflow | `.claude/rules/api-config.md`, `.claude/rules/models.md`, `.claude/rules/testing.md`, `.claude/commands/bisect.md`, `.claude/skills/02-env-source-log/build-and-test.md` | covered |
| `verl` | agent loop design, async server manager, FSDP backend worker patterns | `.claude/skills/03-design/add-workflow.md`, `.claude/skills/03-design/engine-expert.md`, `.claude/agents/fsdp-engine-expert.md` | covered |
| `vllm` | async serving, persistent batch, PD disaggregation and KV transfer | `.claude/skills/01-server/SKILL.md`, `.claude/skills/03-design/model-onboard.md` | covered |

## Placement Decisions

### `rules`

Used for constraints that should persist across tasks:

- config safety
- distributed invariants
- testing discipline
- model structure conventions

### `skills`

Used for executable workflows:

- debugging a CUDA crash
- adding a workflow or model
- profiling and benchmarking
- CI triage and PR workflow

### `knowledge`

Used for deep technical references that are too detailed for a task guide:

- TMA hazard interpretation
- SASS/MMA reading
- NCCL contributor patterns
- kernel race or hang postmortems
- engine integration references
- serving topology references
- rollout contract references

### `agents`

Reserved for domains that require opinionated expert guidance across many codebases:

- FSDP
- Megatron
- MoE engines
- RL algorithms
- launchers/schedulers
- model debug
- serving systems
- CI triage
- distributed runtime correctness
- kernel performance

## Remaining Expert Gaps

These are future-expansion candidates after the current expertization pass:

- deeper `vLLM` scheduler and metrics internals for `serving-systems-expert`
- more explicit `Ollama` local packaging and deployment examples for `serving-systems-expert`
- an `ai-dev-systems-expert` if the cross-harness development system becomes a larger usage slice
- additional `Megatron-LM` multimodal notes if `megatron-engine-expert` needs stronger multimodal guidance
- a future `checkpoint-expert` if save, restore, and refresh logic grows beyond current engine ownership

## Verification Notes

This audit verified:

- all 12 repos are represented
- all canonical expert themes have at least one owned knowledge document
- all critical `.claude` assets found during audit are either absorbed or intentionally generalized elsewhere
- the missing referenced files were created
- underrepresented repo knowledge was mapped into the most appropriate existing categories instead of being left implicit
- expert, knowledge, and workflow layers now have an explicit navigation order

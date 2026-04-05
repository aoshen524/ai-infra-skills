# Expert Knowledge Map

This document is the canonical map from expert themes to owned knowledge in
`ai-infra-skills`.

Mapping rules:

- every file under `knowledge/` has exactly one primary expert owner
- a knowledge document may have zero or more secondary expert consumers
- skills and rules can support an expert, but they are not primary knowledge owners
- `code-verifier` is a utility agent and is not part of the canonical expert ownership map

## Expert Registry

| Expert | Owns | Primary Knowledge | Secondary Knowledge | Typical Tasks | Out of Scope | Workflow / Rules |
| ------ | ---- | ----------------- | ------------------- | ------------- | ------------ | ---------------- |
| `algorithm-expert` | RL algorithm selection, reward shaping, loss behavior | `knowledge/rl/algorithm-patterns.md` | `knowledge/rl/tree-training.md`, `knowledge/rl/workflow-contracts.md` | PPO or GRPO tuning, ratio explosions, reward collapse, clip-rate debugging | launcher issues, kernel tuning, serving topology | `.claude/skills/03-design/add-workflow.md`, `.claude/rules/testing.md` |
| `fsdp-engine-expert` | dense-model sharding and FSDP workflow integration | `knowledge/distributed/fsdp-engine-patterns.md` | `knowledge/distributed/debug-distributed.md`, `knowledge/distributed/nccl-patterns.md` | FSDP layout, wrap policy, checkpoint load, rollout refresh | Megatron PP scheduling, MoE routing internals | `.claude/skills/03-design/engine-expert.md`, `.claude/rules/distributed.md` |
| `megatron-engine-expert` | Megatron PP and hybrid-parallel engine integration | `knowledge/distributed/megatron-engine-patterns.md` | `knowledge/distributed/debug-distributed.md`, `knowledge/distributed/nccl-patterns.md` | PP schedule choice, hybrid TP/PP/EP, shard resume bugs | FSDP wrap policy, generic serving issues | `.claude/skills/03-design/engine-expert.md`, `.claude/rules/distributed.md` |
| `moe-engine-expert` | sparse-engine integration, EP and ETP layouts, router-aware engine concerns | `knowledge/distributed/moe-engine-patterns.md` | `knowledge/rl/tree-training.md`, `knowledge/distributed/debug-distributed.md` | MoE onboarding, expert placement, sparse checkpoint layout, router balance | dense-only FSDP advice, CI-only issues | `.claude/skills/03-design/model-onboard.md`, `.claude/rules/models.md` |
| `launcher-scheduler` | cluster launching, scheduler contracts, resource placement | `knowledge/distributed/launcher-scheduler-patterns.md` | `knowledge/distributed/debug-distributed.md`, `knowledge/serving/gateway-serving-patterns.md` | Slurm or Ray failures, port allocation, worker startup, shared storage issues | RL loss tuning, kernel internals | `.claude/skills/01-server/SKILL.md`, `.claude/rules/distributed.md` |
| `model-debug-agent` | model onboarding and runtime model-debug loops | `knowledge/serving/model-onboarding-debug.md` | `knowledge/serving/disaggregated-serving.md`, `knowledge/distributed/fsdp-engine-patterns.md` | model load mismatch, config translation, world-size assumptions, weight mapping | CI bucket ownership, scheduler policy | `.claude/skills/03-design/model-onboard.md`, `.claude/rules/models.md` |
| `distributed-systems-expert` | cross-rank correctness, collective behavior, NCCL and DTensor debugging | `knowledge/distributed/debug-distributed.md`, `knowledge/distributed/nccl-patterns.md`, `knowledge/hardware/nvidia-datacenter-gpu-matrix.md` | `knowledge/distributed/fsdp-engine-patterns.md`, `knowledge/distributed/megatron-engine-patterns.md`, `knowledge/distributed/moe-engine-patterns.md`, `knowledge/hardware/gpu-spec-reference.md`, `knowledge/hardware/b300-blackwell-notes.md` | hangs, rank mismatch, collective timeout, process-group confusion, interconnect interpretation, GPU-topology assumptions | reward semantics, kernel micro-optimization | `.claude/skills/02-env-source-log/debug-cuda-crash.md`, `.claude/rules/distributed.md` |
| `kernel-performance-expert` | kernel profiling, TMA behavior, JIT variants, low-level GPU debugging | `knowledge/kernels/debug-2cta-hang.md`, `knowledge/kernels/jit-architecture.md`, `knowledge/kernels/sass-mma-analysis.md`, `knowledge/kernels/tma-sync-hazard.md`, `knowledge/kernels/kernel-autoresearch-loop.md` | `knowledge/hardware/gpu-spec-reference.md`, `knowledge/hardware/b300-blackwell-notes.md`, `knowledge/hardware/nvidia-datacenter-gpu-matrix.md` | SASS inspection, TMA hazard diagnosis, JIT cache design, multi-CTA deadlocks, iterative kernel optimization loops, MFU denominator selection | launcher failures, reward tuning, cluster policy | `.claude/skills/05-profiler/SKILL.md`, `.claude/skills/03-design/add-kernel.md` |
| `serving-systems-expert` | serving topology, gateway integration, session ownership, P/D disaggregation | `knowledge/serving/gateway-serving-patterns.md`, `knowledge/serving/disaggregated-serving.md` | `knowledge/serving/model-onboarding-debug.md`, `knowledge/rl/workflow-contracts.md` | gateway serving, online RL serving loop, KV transfer, sticky routing, session state | PPO math, CI bisection | `.claude/skills/01-server/SKILL.md`, `.claude/skills/03-design/agentic-rl-gateway.md` |
| `rl-workflow-expert` | rollout workflow contracts, agent loops, reward wiring, trainable trajectory shape | `knowledge/rl/workflow-contracts.md`, `knowledge/rl/tree-training.md` | `knowledge/serving/gateway-serving-patterns.md`, `knowledge/rl/algorithm-patterns.md` | workflow extension, multi-turn rollout, online RL data path, eval dataset contract | launcher placement, kernel tuning | `.claude/skills/03-design/add-workflow.md`, `.claude/rules/testing.md` |
| `ci-triage-expert` | CI failure bucketing, runner drift, regression bisection | `knowledge/ci-cd/ci-triage-patterns.md`, `knowledge/ci-cd/sglang-ci-system.md` | `knowledge/distributed/debug-distributed.md`, `knowledge/serving/model-onboarding-debug.md` | failing pipeline triage, flaky runner diagnosis, pass/fail boundary, infra vs code split | day-to-day local verification after a small change | `.claude/skills/06-code-review/ci-failure-triage.md`, `.claude/commands/bisect.md` |

## Knowledge Ownership

| Knowledge Path | Primary Owner | Secondary Experts |
| -------------- | ------------- | ----------------- |
| `knowledge/ci-cd/ci-triage-patterns.md` | `ci-triage-expert` | `code-verifier` |
| `knowledge/ci-cd/sglang-ci-system.md` | `ci-triage-expert` | `code-verifier` |
| `knowledge/distributed/debug-distributed.md` | `distributed-systems-expert` | `fsdp-engine-expert`, `megatron-engine-expert`, `moe-engine-expert`, `launcher-scheduler`, `ci-triage-expert` |
| `knowledge/distributed/fsdp-engine-patterns.md` | `fsdp-engine-expert` | `distributed-systems-expert`, `model-debug-agent` |
| `knowledge/distributed/launcher-scheduler-patterns.md` | `launcher-scheduler` | `distributed-systems-expert`, `serving-systems-expert` |
| `knowledge/distributed/megatron-engine-patterns.md` | `megatron-engine-expert` | `distributed-systems-expert`, `moe-engine-expert` |
| `knowledge/distributed/moe-engine-patterns.md` | `moe-engine-expert` | `megatron-engine-expert`, `distributed-systems-expert` |
| `knowledge/distributed/nccl-patterns.md` | `distributed-systems-expert` | `fsdp-engine-expert`, `megatron-engine-expert`, `kernel-performance-expert` |
| `knowledge/hardware/b300-blackwell-notes.md` | `distributed-systems-expert` | `kernel-performance-expert`, `serving-systems-expert` |
| `knowledge/hardware/gpu-spec-reference.md` | `kernel-performance-expert` | `distributed-systems-expert`, `serving-systems-expert` |
| `knowledge/hardware/nvidia-datacenter-gpu-matrix.md` | `distributed-systems-expert` | `kernel-performance-expert`, `serving-systems-expert` |
| `knowledge/kernels/debug-2cta-hang.md` | `kernel-performance-expert` | none |
| `knowledge/kernels/kernel-autoresearch-loop.md` | `kernel-performance-expert` | none |
| `knowledge/kernels/jit-architecture.md` | `kernel-performance-expert` | `serving-systems-expert` |
| `knowledge/kernels/sass-mma-analysis.md` | `kernel-performance-expert` | none |
| `knowledge/kernels/tma-sync-hazard.md` | `kernel-performance-expert` | none |
| `knowledge/rl/algorithm-patterns.md` | `algorithm-expert` | `rl-workflow-expert` |
| `knowledge/rl/tree-training.md` | `rl-workflow-expert` | `algorithm-expert`, `moe-engine-expert` |
| `knowledge/rl/workflow-contracts.md` | `rl-workflow-expert` | `algorithm-expert`, `serving-systems-expert` |
| `knowledge/serving/disaggregated-serving.md` | `serving-systems-expert` | `model-debug-agent` |
| `knowledge/serving/gateway-serving-patterns.md` | `serving-systems-expert` | `launcher-scheduler`, `rl-workflow-expert` |
| `knowledge/serving/model-onboarding-debug.md` | `model-debug-agent` | `serving-systems-expert`, `ci-triage-expert` |

## Utility Agent Note

`code-verifier` remains a utility agent:

- it consumes `knowledge/ci-cd/ci-triage-patterns.md` and
  `knowledge/ci-cd/sglang-ci-system.md`
- it does not own canonical knowledge
- it should be used after edits, while `ci-triage-expert` should be used to reason about
  failing CI systems and regression classes

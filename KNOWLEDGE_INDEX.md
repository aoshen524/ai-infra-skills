# Knowledge Index

Topic-level index for the `knowledge/` tree.

Use this file after `INDEX.md` when the task clearly needs deep reference material but
the exact document is still unclear.

## How to Use

1. find the closest topic below
1. open the primary document first
1. open secondary documents only if the first one does not answer the question
1. return to the matching expert or skill for execution workflow

## Topic Map

| Topic | Primary Docs | Typical Questions | Primary Expert |
| ----- | ------------ | ----------------- | -------------- |
| distributed runtime | `knowledge/distributed/debug-distributed.md`, `knowledge/distributed/nccl-patterns.md` | why does training hang, which collective mismatched, what topology assumption is wrong | `distributed-systems-expert` |
| engine integration | `knowledge/distributed/fsdp-engine-patterns.md`, `knowledge/distributed/megatron-engine-patterns.md`, `knowledge/distributed/moe-engine-patterns.md` | should this model use FSDP or Megatron, where does EP or TP fit, why does sharding or routing fail | `fsdp-engine-expert`, `megatron-engine-expert`, `moe-engine-expert` |
| Megatron memory planning | `knowledge/distributed/megatron-memory-planning.md`, `knowledge/distributed/megatron-engine-patterns.md` | will this config fit, which stage will peak, should TP or PP change first, where should uneven layers land | `megatron-engine-expert` |
| launcher and cluster contracts | `knowledge/distributed/launcher-scheduler-patterns.md` | why did workers fail to start, which env var or resource contract drifted, is this a launcher or scheduler bug | `launcher-scheduler` |
| kernel debugging | `knowledge/kernels/debug-2cta-hang.md`, `knowledge/kernels/tma-sync-hazard.md` | why did the kernel deadlock, is racecheck lying, which barrier or TMA rule was violated | `kernel-performance-expert` |
| kernel design and iteration | `knowledge/kernels/kernel-autoresearch-loop.md`, `knowledge/kernels/jit-architecture.md`, `knowledge/kernels/sass-mma-analysis.md`, `knowledge/kernels/tilelang-kernel-patterns.md` | how should an agent optimize a kernel, how does the JIT path work, what did the compiled MMA path become, when is TileLang the right DSL | `kernel-performance-expert` |
| hardware grounding | `knowledge/hardware/nvidia-datacenter-gpu-matrix.md`, `knowledge/hardware/gpu-spec-reference.md`, `knowledge/hardware/b300-blackwell-notes.md` | which GPU family should be the baseline, how should MFU be interpreted, is this a new-hardware software-readiness issue | `distributed-systems-expert`, `kernel-performance-expert` |
| RL algorithms | `knowledge/rl/algorithm-patterns.md` | why is PPO unstable, which reward or ratio signal is wrong, what algorithm family fits this task | `algorithm-expert` |
| RL workflow contracts | `knowledge/rl/workflow-contracts.md`, `knowledge/rl/tree-training.md`, `knowledge/rl/slime-repo-navigation.md`, `knowledge/rl/dataset-pipeline-contracts.md` | how should rollout and reward responsibilities split, how does multi-turn or tree training affect data shape, where should I look in SLIME first, what dataset and sampler contracts must stay stable | `rl-workflow-expert` |
| serving systems | `knowledge/serving/gateway-serving-patterns.md`, `knowledge/serving/disaggregated-serving.md`, `knowledge/serving/torch-compile-coverage.md`, `knowledge/serving/cuda-graph-patterns.md` | how should sessions and gateways work, when should P/D be used, why did a fast kernel regress E2E serving, when is CUDA Graph safe and worthwhile | `serving-systems-expert` |
| model onboarding | `knowledge/serving/model-onboarding-debug.md` | why does the model fail to load, which registry or config translation is broken, is this a load or sharding issue | `model-debug-agent` |
| CI and regressions | `knowledge/ci-cd/ci-triage-patterns.md`, `knowledge/ci-cd/sglang-ci-system.md` | is this flake or code regression, how should the failure be bucketed, which workflow or runner changed | `ci-triage-expert` |

## Symptom Routing

If you are starting from a symptom instead of a topic:

| Symptom | Open First | Then |
| ------- | ---------- | ---- |
| distributed hang or timeout | `knowledge/distributed/debug-distributed.md` | `knowledge/distributed/nccl-patterns.md` |
| wrong MFU, bandwidth, or scaling denominator | `knowledge/hardware/nvidia-datacenter-gpu-matrix.md` | `knowledge/hardware/gpu-spec-reference.md` |
| Megatron config might not fit memory | `knowledge/distributed/megatron-memory-planning.md` | `knowledge/distributed/megatron-engine-patterns.md` |
| new GPU behaves oddly on software stack | `knowledge/hardware/b300-blackwell-notes.md` | `knowledge/hardware/nvidia-datacenter-gpu-matrix.md` |
| kernel deadlock or sanitizer confusion | `knowledge/kernels/debug-2cta-hang.md` | `knowledge/kernels/tma-sync-hazard.md` |
| need a safe agentic kernel optimization loop | `knowledge/kernels/kernel-autoresearch-loop.md` | `knowledge/kernels/sass-mma-analysis.md` |
| need TileLang-specific kernel guidance | `knowledge/kernels/tilelang-kernel-patterns.md` | `knowledge/kernels/kernel-autoresearch-loop.md` |
| serving path needs CUDA Graph support | `knowledge/serving/cuda-graph-patterns.md` | `knowledge/serving/torch-compile-coverage.md` |
| serving got slower after custom op integration | `knowledge/serving/torch-compile-coverage.md` | `knowledge/serving/disaggregated-serving.md` |
| gateway, session, or online RL serving design | `knowledge/serving/gateway-serving-patterns.md` | `knowledge/rl/workflow-contracts.md` |
| model loads fail or configs do not translate | `knowledge/serving/model-onboarding-debug.md` | `knowledge/distributed/fsdp-engine-patterns.md` |
| dataset packing or sampler resume feels unreliable | `knowledge/rl/dataset-pipeline-contracts.md` | `knowledge/rl/workflow-contracts.md` |
| reward loop or rollout semantics feel wrong | `knowledge/rl/workflow-contracts.md` | `knowledge/rl/algorithm-patterns.md` |
| need the fastest route inside the SLIME repo | `knowledge/rl/slime-repo-navigation.md` | `knowledge/rl/workflow-contracts.md` |
| CI failure or runner drift | `knowledge/ci-cd/ci-triage-patterns.md` | `knowledge/ci-cd/sglang-ci-system.md` |

## Reading Heuristics

- open one primary doc before opening an entire folder
- prefer the document owned by the closest expert rather than the longest document
- use hardware notes to interpret distributed and kernel results, not in isolation
- use serving and RL docs together only when the system truly crosses both boundaries

## Adjacent Entry Points

- `INDEX.md` for top-level routing
- `EXPERT_KNOWLEDGE_MAP.md` for ownership and canonical expert mapping
- `.claude/agents/*` for judgment-heavy routing
- `.claude/skills/*` for execution workflows

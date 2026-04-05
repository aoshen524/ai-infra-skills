# AI Infrastructure Skills for Claude Code

A collection of professional Claude Code skills, agents, and knowledge bases for 12 major open-source AI infrastructure frameworks.

## Fast Start

If you only open one file first, open [INDEX.md](/home/ubuntu/scipts/verl-grounding/ai-infra-skills/INDEX.md).

Recommended navigation order:

1. [INDEX.md](/home/ubuntu/scipts/verl-grounding/ai-infra-skills/INDEX.md)
1. [EXPERT_KNOWLEDGE_MAP.md](/home/ubuntu/scipts/verl-grounding/ai-infra-skills/EXPERT_KNOWLEDGE_MAP.md)
1. matching `.claude/skills/*` guide
1. matching `knowledge/*` document

## Target Frameworks

| Domain | Projects |
|--------|----------|
| Inference | vLLM, SGLang, Ollama, TensorRT-LLM |
| Training | Megatron-LM, PyTorch/torchtitan |
| RL | veRL, AReaL, SLIME |
| Kernels | FlashAttention, FlashInfer |
| Communication | NCCL |

## Directory Structure

```
ai-infra-skills/
├── .claude/
│   ├── rules/                      # Behavioral constraints (auto-loaded)
│   │   ├── api-config.md           # API and config conventions
│   │   ├── code-style.md           # Code style patterns
│   │   ├── distributed.md          # Distributed training patterns
│   │   ├── models.md               # Model code conventions
│   │   └── testing.md              # Testing strategy
│   │
│   ├── skills/                     # Interactive skills (invoked by user or triggered)
│   │   ├── 01-server/              # Container dev, GPU mgmt, cluster setup
│   │   ├── 02-env-source-log/      # Build, CI, CUDA crash debugging
│   │   ├── 03-design/              # Feature evaluation, architecture design
│   │   ├── 04-review-plan/         # Plan review, domain classification
│   │   ├── 05-profiler/            # Benchmarking, NCU analysis, tuning
│   │   └── 06-code-review/         # PR workflow, CI failure triage
│   │
│   ├── agents/                     # Domain experts plus utility agents
│   │   ├── algorithm-expert.md     # RL algorithm behavior and tuning
│   │   ├── fsdp-engine-expert.md   # FSDP/FSDP2 configuration
│   │   ├── megatron-engine-expert.md # Pipeline/hybrid parallelism
│   │   ├── moe-engine-expert.md    # MoE / EP / ETP / grouped experts
│   │   ├── distributed-systems-expert.md # NCCL, ranks, process groups
│   │   ├── kernel-performance-expert.md  # TMA, SASS, JIT, deadlocks
│   │   ├── serving-systems-expert.md     # Gateways, P/D serving, KV transfer
│   │   ├── rl-workflow-expert.md   # Rollout contracts and trajectory shape
│   │   ├── ci-triage-expert.md     # CI bucketing and bisection
│   │   ├── launcher-scheduler.md   # Cluster launching (Slurm/Ray/K8s)
│   │   ├── model-debug-agent.md    # Model onboarding debugging
│   │   └── code-verifier.md        # Utility verification agent
│   │
│   └── commands/                   # Slash commands
│       ├── commit.md               # /commit — Conventional Commits
│       ├── review-pr.md            # /review-pr — Dynamic PR review
│       ├── create-pr.md            # /create-pr — Rebase + squash + PR
│       └── bisect.md               # /bisect — Git bisect regression
│
├── INDEX.md                        # Fastest task-routing entrypoint
├── COVERAGE_AUDIT.md               # 12-repo coverage matrix and placement audit
├── EXPERT_KNOWLEDGE_MAP.md         # Canonical expert-to-knowledge ownership map
├── meta/                           # Historical or maintenance-only docs
└── knowledge/                      # Deep technical references
    ├── distributed/
    │   ├── debug-distributed.md    # Distributed training debugging
    │   ├── nccl-patterns.md        # NCCL development patterns
    │   ├── fsdp-engine-patterns.md # FSDP engine integration reference
    │   ├── megatron-engine-patterns.md # Megatron engine reference
    │   ├── moe-engine-patterns.md  # MoE engine integration reference
    │   └── launcher-scheduler-patterns.md # Cluster launch contracts
    ├── kernels/
    │   ├── debug-2cta-hang.md      # 2CTA kernel hang debugging
    │   ├── kernel-autoresearch-loop.md # Constrained iterative kernel optimization
    │   ├── tma-sync-hazard.md      # TMA racecheck false positives
    │   ├── sass-mma-analysis.md    # SASS/HGMMA instruction analysis
    │   └── jit-architecture.md     # JIT compilation patterns
    ├── hardware/
    │   ├── gpu-spec-reference.md   # What hardware facts matter for AI infra
    │   ├── nvidia-datacenter-gpu-matrix.md # A100/H100/H200/B200/B300 decision matrix
    │   └── b300-blackwell-notes.md # New-hardware bring-up and software maturity notes
    ├── rl/
    │   ├── algorithm-patterns.md   # PPO-family behavior and tuning
    │   ├── workflow-contracts.md   # Workflow / rollout / reward contracts
    │   └── tree-training.md        # Prefix sharing and tree attention patterns
    ├── serving/
    │   ├── gateway-serving-patterns.md # Gateway + session + reward writeback
    │   ├── disaggregated-serving.md    # Prefill/decode separation and KV transfer
    │   ├── torch-compile-coverage.md   # Graph-break and custom-op integration checks
    │   └── model-onboarding-debug.md   # Model load and conversion debug loop
    └── ci-cd/
        ├── ci-triage-patterns.md   # Failure bucketing and bisection
        └── sglang-ci-system.md     # SGLang CI system overview
```

## Skill Categories

### 01-server: Remote Server & Cluster Management
Container-based development, remote GPU backend usage, GPU resource monitoring, multi-node cluster launching, and session management. Essential for any GPU-accelerated project.

### 02-env-source-log: Environment, Build & Debug
Build/install workflows, CI/CD pipeline patterns, CUDA crash debugging with API logging, compute-sanitizer, and kernel printf.

### 03-design: Architecture & Design
Feature evaluation frameworks, module design patterns, model onboarding workflows, gateway-based agentic RL integration, engine expert guidance for inference and training systems.

### 04-review-plan: Plan & Review
Structured implementation planning with domain classification, risk assessment, and phased execution. Domain-specific review checklists.

### 05-profiler: Performance Profiling & Tuning
Kernel benchmarking (CUPTI/Events), SM90 block size tuning, SASS/MMA analysis, auto-benchmark workflows, and two-phase Chrome trace triage.

### 06-code-review: Code Review & CI
PR workflows with dynamic agent allocation, Conventional Commits, CI failure triage, git bisect for regressions.

## Rules (Auto-Loaded)

Rules in `.claude/rules/` are automatically loaded and enforced:
- **api-config**: Dataclass conventions, field ordering, validation, backward compatibility
- **code-style**: Naming, performance patterns (no GPU-CPU sync in hot paths), logging, imports
- **distributed**: Process groups, DeviceMesh/DTensor, communication patterns, common pitfalls
- **models**: Minimal model files, audit all variants, unify shared components, checkpoint compat
- **testing**: Pytest markers, GPU test constraints, mocking distributed, assertions

## Expert-Owned Knowledge

`knowledge/` is now organized as an expert-owned reference layer.

- every knowledge document has one primary expert owner
- domain experts point to their canonical knowledge before pointing to skills
- skills describe execution workflows, while knowledge captures stable design and debug context
- `code-verifier` remains a utility agent rather than a canonical knowledge owner
- hardware references exist to ground MFU, roofline, bandwidth, and software-support claims
- concrete NVIDIA data center GPU notes now exist for A100, H100, H200, B200, and B300-era decisions

See [EXPERT_KNOWLEDGE_MAP.md](/home/ubuntu/scipts/verl-grounding/ai-infra-skills/EXPERT_KNOWLEDGE_MAP.md) for the source-of-truth mapping.

## Expert-First Navigation

Use this order when navigating the repo:

1. open `INDEX.md` if task routing is still unclear
1. find the closest expert for the problem domain
1. read the expert's owned knowledge documents
1. run the matching skill or command for workflow execution
1. use `.claude/rules/` as the final constraint and review layer

## Skill Design Principles

The skill system in this repo follows a few hard defaults:

- skills should stay small and task-specific rather than becoming giant prompts
- category `SKILL.md` files should route, not duplicate every child guide
- deep background should move into `knowledge/`
- expert agents should own judgment-heavy domain routing
- gotchas should be written down explicitly when failure modes repeat

See [SKILL_AUTHORING.md](/home/ubuntu/scipts/verl-grounding/ai-infra-skills/SKILL_AUTHORING.md) for the repo's skill-writing and maintenance conventions.

## Usage

### As a Git Submodule
```bash
git submodule add <repo-url> ai-infra-skills
```

### In Claude Code Settings
Add to your project's `.claude/settings.json`:
```json
{
  "additionalWorkingDirectories": [
    "ai-infra-skills/.claude/skills/03-design",
    "ai-infra-skills/.claude/skills/04-review-plan",
    "ai-infra-skills/.claude/agents"
  ]
}
```

## Sources

Content is synthesized and generalized from the following projects:
- FlashAttention / FlashInfer — kernel debugging, benchmarking, SASS analysis
- SGLang — CI system, auto-benchmark, CUDA crash debugging
- SGLang field workflows — remote GPU backend usage, auto-driven benchmark search, torch profiler triage, and torch.compile coverage checks
- AReaL — PR review, agents, algorithms, distributed debugging, MoE engine patterns
- AReaL — proxy gateway integration, online RL session design, tree training
- AReaL — launcher/scheduler contracts and AI-assisted development structure
- TensorRT-LLM — CI failure triage, model debugging
- torchtitan — git bisect workflow
- NCCL — development patterns, code style
- Megatron-LM — container dev, uv, API compatibility, CI policy
- vLLM — async serving, persistent batch, disaggregated P/D serving
- Ollama — model packaging, runtime troubleshooting
- veRL — agent loop and hybrid training-serving workflow patterns
- SLIME — reward/rollout/eval dataset extension patterns
- Liger-Kernel — kernel benchmark structure, parity-first performance testing
- sparse-mask-attention — constrained kernel autoresearch loop and optimization logging
- b300-benchmarks — hardware spec grounding, interconnect validation, and new-hardware software support notes
- NVIDIA data center GPU documentation — current generation memory, NVLink, and Blackwell compatibility grounding
- recent SGLang workflow writeups — remote GPU backend usage, auto-driven benchmark loops, CUDA crash artifact capture, and trace-to-action profiler triage

See [COVERAGE_AUDIT.md](/home/ubuntu/scipts/verl-grounding/ai-infra-skills/COVERAGE_AUDIT.md) for the repo-to-location coverage matrix.

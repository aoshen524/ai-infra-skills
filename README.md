# AI Infrastructure Skills

`ai-infra-skills` is a curated skill, expert, and knowledge repository for Claude Code and
Codex agents working on AI infrastructure systems.

It is organized around a simple runtime order:

1. route the task with `INDEX.md`
1. find the right domain owner in `EXPERT_KNOWLEDGE_MAP.md`
1. read the owned `knowledge/*` docs
1. execute with `.claude/skills/*` or `.claude/commands/*`
1. validate against `.claude/rules/*`

If you only open one file first, open [INDEX.md](INDEX.md).

## What Is In This Repo

- domain experts for distributed systems, serving, kernels, RL workflows, engines, and CI
- knowledge docs for stable technical references
- executable skills for common implementation, profiling, review, and debugging workflows
- slash commands for repeatable agent loops such as `/solo`, `/review-pr`, and `/bisect`
- supporting communication skills for technical diagrams and architecture decks

## Structure

```text
ai-infra-skills/
├── README.md
├── INDEX.md
├── CLAUDE.md
├── KNOWLEDGE_INDEX.md
├── EXPERT_KNOWLEDGE_MAP.md
├── .claude/
│   ├── agents/
│   ├── commands/
│   ├── rules/
│   └── skills/
│       ├── 01-server/
│       ├── 02-env-source-log/
│       ├── 03-design/
│       ├── 04-review-plan/
│       ├── 05-profiler/
│       ├── 06-code-review/
│       └── 07-communication/
└── knowledge/
    ├── distributed/
    ├── hardware/
    ├── kernels/
    ├── rl/
    ├── serving/
    └── ci-cd/
```

## What Was Integrated From `infra-skills`

This repo now absorbs the most reusable parts of
[`yzlnew/infra-skills`](https://github.com/yzlnew/infra-skills) into the current
expert-owned architecture rather than keeping them as a separate parallel skill tree.

Integrated themes include:

- TileLang kernel development patterns
- Megatron memory estimation and planning workflows
- SLIME repo navigation and workflow-extension routing
- technical communication helpers for TikZ flowcharts and Material You slides

These were mapped into `knowledge/*`, `.claude/skills/*`, and runtime indexes so they are
discoverable from the same entrypoints as the rest of the repo.

## Quick Start

### For Agents

- open `INDEX.md`
- route to the nearest expert or skill
- use `KNOWLEDGE_INDEX.md` if you need topic-level reference routing
- use `CLAUDE.md` when you need the full repository map

### For Humans

- start with `README.md` for the repo purpose and credits
- use `CLAUDE.md` to understand the complete structure
- use `INDEX.md` when you want the shortest path to the right asset

## References And Acknowledgements

This repository is a synthesis project. It generalizes patterns from many open-source AI
infra codebases and workflow writeups into reusable agent-facing assets.

Core framework and system references include:

- [AReaL](https://github.com/inclusionAI/AReaL)
- [Megatron-LM](https://github.com/NVIDIA/Megatron-LM)
- [TensorRT-LLM](https://github.com/NVIDIA/TensorRT-LLM)
- [FlashAttention](https://github.com/Dao-AILab/flash-attention)
- [FlashInfer](https://github.com/flashinfer-ai/flashinfer)
- [NCCL](https://github.com/NVIDIA/nccl)
- [Ollama](https://github.com/ollama/ollama)
- [SGLang](https://github.com/sgl-project/sglang)
- [SLIME](https://github.com/THUDM/slime)
- [veRL](https://github.com/volcengine/verl)
- [vLLM](https://github.com/vllm-project/vllm)
- [torchtitan](https://github.com/pytorch/torchtitan)

Additional supporting references include:

- [TileLang](https://github.com/tile-ai/tilelang)
- [Liger-Kernel](https://github.com/linkedin/Liger-Kernel)
- [sparse-mask-attention](https://github.com/xgbj/sparse-mask-attention)
- [b300-benchmarks](https://github.com/ricardojacomini/b300-benchmarks)
- [megatron_memory_estimator](https://huggingface.co/spaces/ISEEKYAN/megatron_memory_estimator)

Special thanks to:

- the maintainers and contributors of all referenced repositories
- the maintainers of SGLang field workflow writeups that informed the auto-driven
  benchmark, crash-debugging, and profiler triage flows
- the author and maintainer of [`yzlnew/infra-skills`](https://github.com/yzlnew/infra-skills)
- [`yzlnew/infra-skills`](https://github.com/yzlnew/infra-skills) for the TileLang,
  Megatron memory planning, SLIME navigation, and technical-communication skill material
  that was adapted into this repo's expert-first layout

## Notes

- `CLAUDE.md` remains the authoritative repository map for runtime use
- `README.md` is the GitHub-facing entrypoint and project overview
- `COVERAGE_AUDIT.md` tracks how source repos and external references were absorbed

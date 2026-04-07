---
name: design-architecture
description: Feature evaluation, architecture analysis, and module design patterns for AI infrastructure projects (inference engines, training frameworks, kernel libraries).
tools:
  - Bash
  - Read
  - Write
  - Grep
  - Glob
  - Agent
---

# Design & Architecture Skills

Patterns and workflows for designing and implementing new features in AI infrastructure codebases: inference engines, RL training frameworks, and CUDA kernel libraries.

## Included Guides

| Guide | When to Use |
|-------|-------------|
| [model-onboard.md](model-onboard.md) | Adding a new model to an inference engine or serving framework |
| [cuda-graph-integration.md](cuda-graph-integration.md) | Adding or debugging CUDA Graph in a serving path |
| [add-kernel.md](add-kernel.md) | Implementing a new CUDA kernel (JIT or AOT) |
| [tilelang-kernel.md](tilelang-kernel.md) | Implementing or debugging a TileLang-based kernel |
| [dataset-pipeline.md](dataset-pipeline.md) | Designing cached, packable, resume-safe dataset pipelines |
| [add-workflow.md](add-workflow.md) | Adding an RL workflow, reward function, or rollout function |
| [slime-workflows.md](slime-workflows.md) | Navigating and extending SLIME workflows and examples |
| [megatron-memory-estimation.md](megatron-memory-estimation.md) | Planning Megatron TP/PP/EP/CP fits before launch |
| [agentic-rl-gateway.md](agentic-rl-gateway.md) | Integrating external agents or human sessions into an RL loop through a proxy gateway |
| [engine-expert.md](engine-expert.md) | Choosing and configuring distributed training engines and RL algorithms |

## General Principles

1. **Survey before writing** -- always check for existing implementations that can be reused or extended before creating new files.
2. **Gate-checked phases** -- break complex features into phases with validation gates between them. Do not proceed to the next phase until the current one passes.
3. **Hierarchical testing** -- test bottom-up: unit (single op/block) then layer then end-to-end. Catch bugs at the smallest scope.
4. **Minimal implementation** -- implement only what is needed. Do not add configurability, metrics, or abstractions beyond the request scope.
5. **Self-contained modules** -- each new module should be independently auditable. Avoid hidden coupling through cross-imports between peer modules.

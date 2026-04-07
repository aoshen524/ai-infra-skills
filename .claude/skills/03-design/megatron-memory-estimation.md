# Megatron Memory Estimation

Use this guide when the task is to estimate whether a Megatron-style training
configuration fits on the target GPU fleet before launching a real job.

## When To Use

Use it for:

- planning TP, PP, EP, or CP strategy
- checking whether a model fits A100, H100, H200, B200, or B300-class GPUs
- comparing memory tradeoffs between dense and MoE configs
- reasoning about uneven pipeline stage placement

Do not use it for:

- runtime deadlock or NCCL debugging
- throughput tuning after the model already fits
- generic FSDP-only memory questions

## Workflow

1. collect the real model config:
   - layer count
   - hidden size
   - attention structure
   - FFN size
   - vocab size
   - MoE fields if relevant
1. collect runtime assumptions:
   - sequence length
   - micro-batch size
   - target GPU family
   - candidate TP, PP, EP, CP settings
1. read `knowledge/distributed/megatron-memory-planning.md`
1. estimate the baseline first
1. change one parallelism dimension at a time
1. record the peak stage and the next mitigation if it does not fit

## Decision Order

If the model does not fit:

1. increase TP until communication tradeoffs get unattractive
1. add EP for MoE-heavy models
1. add PP for depth-driven pressure
1. enable activation recomputation if activations still dominate
1. reduce micro-batch size as the clean final lever

## What A Good Answer Should Include

- exact config assumptions
- peak memory or peak stage
- which dimension dominates memory
- whether the config fits the target GPU budget
- the best next adjustment if it does not fit

## Use With

- `knowledge/distributed/megatron-memory-planning.md`
- `knowledge/distributed/megatron-engine-patterns.md`
- `knowledge/hardware/nvidia-datacenter-gpu-matrix.md`
- `.claude/agents/megatron-engine-expert.md`

# Megatron Memory Planning

Use this document when deciding whether a Megatron-style training configuration fits a
target GPU budget before running expensive distributed jobs.

## What This Is For

Memory planning is not only about "does the model fit". It should answer:

- which parallel dimensions should be active
- which stage is likely to be the peak-memory stage
- whether the bottleneck is parameters, optimizer state, or activations
- which mitigation to try first without blindly rewriting the config

## Planning Inputs

Collect these fields before estimating memory:

- layer count
- hidden size
- attention-head structure
- FFN size
- vocab size
- sequence length
- micro-batch size
- whether the model is dense or MoE
- active parallel dimensions: TP, PP, EP, CP, and optional expert TP

If any of those are unknown, the estimate is usually too weak to drive launch decisions.

## Parallelism Effects

### Tensor Parallelism

Use TP first when width is the main problem:

- good for large hidden sizes
- reduces per-rank weight footprint
- increases communication pressure

TP is often the cleanest first lever, but it stops scaling gracefully once communication
dominates.

### Pipeline Parallelism

Use PP when depth is the main reason the model does not fit:

- splits layers across stages
- creates pipeline bubbles that must be paid for elsewhere
- shifts memory pressure unevenly across stages

PP should be planned together with micro-batch count. A PP decision without a micro-batch
decision is incomplete.

### Expert Parallelism

Use EP for MoE models:

- reduces expert-state memory per rank
- usually gives the largest MoE-specific memory win
- can create load-balance and routing sensitivity

If the model is MoE and experts dominate, EP is often a better lever than adding more PP.

### Context Parallelism

Use CP when long context makes activations the limiting factor:

- reduces sequence-dimension activation pressure
- helps most when sequence length is already large
- only matters if activations, not static model state, are the problem

## Uneven Pipeline Stage Placement

When layer count does not divide evenly across PP stages, treat stage placement as a memory
decision rather than a cosmetic one.

Default heuristic:

- place the extra layer in the last pipeline stage before placing it in the first stage

Why:

- the first stage often carries higher dynamic pressure because its early activations live
  longer through the pipeline
- the last stage is frequently a safer place to absorb one extra layer

Exception:

- if the first stage is materially lighter in static memory because of architecture
  asymmetry, measure before assuming the last stage is always best

## Fit Workflow

1. start from the real model config, not a rough parameter-count guess
1. estimate the baseline with minimal parallelism that respects the hardware topology
1. identify whether peak memory is driven by:
   - weights and gradients
   - optimizer state
   - activations
1. change one parallelism dimension at a time
1. only then add memory-saving features such as activation recomputation

This order makes it easier to explain why the chosen strategy fits.

## Recommended Optimization Order

If the model does not fit:

1. increase TP until communication becomes unattractive
1. add or increase EP for MoE models
1. increase PP if depth still dominates
1. enable activation recomputation if activations are still too large
1. reduce micro-batch size as the last straightforward lever

Do not jump directly to the smallest batch size unless the user explicitly wants a
throughput sacrifice.

## Common Failure Modes

| Symptom | Likely Cause | First Fix |
| ------- | ------------ | --------- |
| estimate says it fits but launch OOMs immediately | stage-local imbalance was ignored | inspect the peak stage, not only the average |
| PP config looks valid but first stage OOMs | extra layers or long-lived activations landed on stage 0 | move the extra layer later or reduce micro-batch size |
| MoE still does not fit after TP and PP | experts dominate memory | increase EP before pushing PP further |
| long-context run OOMs despite strong static fit | activations dominate | add CP or recomputation |
| user compares fits across GPUs incorrectly | hardware capacity and interconnect assumptions were mixed | ground the decision in the target GPU matrix first |

## What To Record In A Planning Report

Always preserve:

- the model source or config path
- active parallel dimensions
- per-stage or peak-rank memory
- sequence length and micro-batch assumptions
- whether recomputation is enabled
- the recommended next adjustment if the fit is marginal

Without these fields, later comparisons between configs become hard to trust.

## Relationship To Other Repo Guides

Use this document with:

- `.claude/skills/03-design/megatron-memory-estimation.md` for the execution workflow
- `knowledge/distributed/megatron-engine-patterns.md` for integration behavior
- `knowledge/hardware/nvidia-datacenter-gpu-matrix.md` for actual GPU-capacity grounding
- `.claude/agents/megatron-engine-expert.md` when PP, EP, or checkpoint layout decisions interact

## Review Questions

1. what actually dominates memory in this configuration?
1. is the planned peak stage explicit?
1. was PP chosen for depth, or only because TP was exhausted?
1. is the recommendation grounded in the real target GPU capacity and topology?

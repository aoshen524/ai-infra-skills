# Dataset Pipeline Design

Use this guide when the task is to add, debug, or review a training-data pipeline rather
than the model architecture itself.

## When To Use

Use it for:

- defining sample schemas
- building cached tokenization flows
- adding packing or sampler logic
- validating resume-safe dataloaders
- multi-dataset mixing and weighting

Do not use it for:

- model-layer implementation
- serving request routing
- one-off dataset format conversion with no lasting pipeline impact

## Workflow

1. define the sample contract first
1. read `knowledge/rl/dataset-pipeline-contracts.md`
1. make tokenization identity and cache invalidation explicit
1. decide whether packing is dynamic, deterministic, or preset
1. define sampler state from global progress, not hidden local iterators
1. validate resume behavior before scaling

## What To Lock Early

- sample schema
- dataset identity
- tokenization function and cache key
- packing mode
- sampler ordering semantics
- resume state shape

## Use With

- `knowledge/rl/dataset-pipeline-contracts.md`
- `knowledge/rl/workflow-contracts.md`
- `.claude/skills/03-design/add-workflow.md`
- `.claude/agents/rl-workflow-expert.md`

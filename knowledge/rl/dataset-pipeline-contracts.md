<!-- source: XTuner V1 dataset architecture rules -->

# Dataset Pipeline Contracts

Use this document when the hard part of a training or RL workflow is no longer the model
itself, but the dataset, packing, sampling, caching, and resume behavior around it.

## Why This Matters

Many training systems fail not because the optimizer is wrong, but because the data
pipeline silently changes:

- sample schema drifts
- packing becomes non-deterministic
- sampler resume mismatches global progress
- tokenization caches go stale or become ambiguous

The more scalable the system becomes, the more important these contracts are.

## Core Sample Contract

Every dataset layer should communicate through one explicit sample protocol.

A minimal useful contract looks like:

```text
input_ids: token ids for the model input
labels: training targets aligned to input_ids
num_tokens: token count used for packing and sampling logic
```

If the pipeline supports RL, multimodal, or tool-using extensions, that base contract may
grow, but it should still remain explicit and versionable.

The critical rule is simple:

- every downstream packer, sampler, and collator should know exactly what one sample is

## Tokenization Contract

Tokenization should be treated as a cachable, hashable transformation, not an ad hoc
pre-step.

Good tokenization contracts preserve:

- tokenizer identity
- source-data identity
- tokenization-function identity
- key options that affect the output

If these inputs change, the cache key should change too.

## Dataset Storage Contract

For large JSONL or similar file-based datasets, strong defaults are:

- one shared read path per file
- an offset table or index for O(1) access
- optional chunk expansion for long-text workflows

This matters because large-scale training pipelines cannot afford expensive line scans or
ambiguous sampling semantics every time they restart.

## Packing Contract

Packing should be explicit about what it returns.

Two stable patterns are:

### Static or Hard Packing

- pack structure computed deterministically from seed or config
- one returned pack may contain multiple source samples
- useful when training throughput depends on stable packing efficiency

### Preset Packing

- pack layout loaded from an explicit artifact
- useful when the exact composition of packs must be reproduced across runs

The key invariant is:

- pack construction should be reproducible from declared state, not hidden runtime behavior

## Sampler Contract

A robust sampler must make these things explicit:

- ordering semantics
- length grouping or other bucket logic
- world-size interaction
- how global progress maps back to local rank state

If the sampler participates in resume, the state must be restorable from explicit counters
instead of implicit iterator position.

## Resume Contract

Resume should preserve:

- sampler state
- dataset state when applicable
- consumed-sample progress at the global level

The important design choice is:

- save global progress and derive per-rank position from it

That is usually safer than trying to restore a pile of local iterator internals.

## Cache Contract

Caching is only valuable when invalidation is trustworthy.

Good cache design should tie entries to:

- data source hash
- tokenize function hash
- tokenizer hash
- relevant parameters that change chunking or truncation behavior

Otherwise the pipeline becomes fast but untrustworthy.

## Multi-Dataset Contract

When mixing datasets, the pipeline should make explicit:

- dataset identity
- sample ratio or weighting
- whether packing is per-dataset or global
- how aliasing and lookup work during resume and audit

Multi-dataset training gets unstable quickly when these decisions are implicit.

## Design Principles

The strongest recurring principles are:

1. protocol-based sample exchange
1. cache-aware tokenization
1. deterministic packing and sampling
1. resume from explicit global state
1. composable pipeline stages instead of one giant opaque loader

## Common Failure Modes

| Symptom | Likely Cause | First Fix |
| ------- | ------------ | --------- |
| resumed run diverges from the original schedule | sampler state was local or ambiguous | store global consumed-sample progress and derive local state |
| cached tokens do not match current code | cache key missed function or tokenizer identity | include transform and tokenizer hashes in cache identity |
| packing changes silently across restarts | pack structure depends on undeclared runtime state | make packing deterministic from seed or artifact |
| long-text chunking drops or duplicates content | chunk boundaries were not part of the contract | preserve explicit chunk metadata and chunk-specific tokenization inputs |
| mixed datasets drift in ratio over time | sample weighting is implicit | make ratio and dataset identity explicit in config and audit output |

## Relationship To Other Repo Guides

Use this document with:

- `.claude/skills/03-design/dataset-pipeline.md`
- `knowledge/rl/workflow-contracts.md`
- `.claude/skills/03-design/add-workflow.md`
- `.claude/agents/rl-workflow-expert.md`

## Review Questions

1. is the sample protocol explicit and stable?
1. can the cache be trusted after code or tokenizer changes?
1. is packing deterministic and auditable?
1. can sampler and dataset state be resumed from explicit global progress?

<!-- source: AReaL -->

# FSDP Engine Patterns

## What This Covers

This document captures the stable patterns behind FSDP or FSDP2 engine integration in
large-model training systems.

Use it for:

- dense-model sharding strategy
- wrap policy and memory strategy
- checkpoint and weight-refresh contracts
- workflow integration for actor, critic, and rollout consumers

## Configuration Layers

Healthy FSDP systems usually separate four surfaces:

- training config
- parallel layout
- FSDP-specific memory and wrap policy
- model load and checkpoint format

Keep these layers distinct. Most hard-to-debug failures come from mixing them into one
flat config surface.

## Parallel Layout Rules

1. make active dimensions multiply to `world_size`
1. keep repeated blocks as the normal wrap boundary
1. decide TP or CP before finalizing FSDP unit size
1. validate checkpoint format against the chosen shard layout

## Memory Strategy

The main levers are:

- activation checkpointing
- mixed precision
- CPU offload
- memory-efficient loading

Pick memory-efficient load paths first. If full weights are materialized before sharding,
later FSDP tuning will not save startup memory.

## Workflow Integration

Common roles:

- actor or policy engine
- critic or value engine
- SFT engine
- reward or preference engine

All of them need explicit answers for:

- how weights are loaded
- how refreshed weights reach rollout or eval workers
- which checkpoint format is expected on resume

## Failure Modes

| Symptom | Likely cause | First check |
| ------- | ------------ | ----------- |
| startup OOM | full weights exist before sharding | inspect load path and init dtype |
| step OOM | wrap boundary or AC policy is wrong | inspect peak memory per repeated block |
| bad resume | shard format mismatch | compare expected full-state vs sharded-state path |
| refresh fails | rollout and trainer disagree on format | inspect checkpoint adapter or broadcast contract |
| poor scaling | over-sharding or sync overhead | compare comm time with compute time |

## Review Questions

1. are active parallel dimensions and rank layout explicit?
1. is the wrap unit aligned with the repeated transformer block?
1. can the load path avoid materializing the full dense model on every rank?
1. is weight refresh topology-aware and restart-safe?

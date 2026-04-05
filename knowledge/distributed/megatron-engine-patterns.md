<!-- source: AReaL, Megatron-LM -->

# Megatron Engine Patterns

## What This Covers

This document captures the durable patterns behind Megatron-style training engines with
pipeline and hybrid parallelism.

Use it for:

- PP schedule selection
- TP, PP, DP, CP, and EP composition
- stage-local checkpoint behavior
- rollout and eval consumers that depend on sharded Megatron weights

## Core Principle

Megatron systems are coordination systems first, math systems second.

You are rarely debugging one dimension in isolation. Most failures come from an
interaction between:

- stage placement
- microbatch count
- checkpoint layout
- inter-stage communication

## Configuration Rules

1. make all active dimensions multiply to `world_size`
1. size PP and microbatches together
1. treat CP or EP as second-order additions after PP basics work
1. validate checkpoint compatibility across all stage and tensor groups

## Schedule Guidance

Typical choices:

- single-stage fallback for smaller models
- 1F1B for straightforward PP
- interleaved schedules when bubble reduction matters and operational complexity is acceptable

Do not choose a fancy schedule before you can explain how startup, resume, and metrics
behave for each stage.

## Integration Points

Megatron-style engines usually need explicit handling for:

- stage or shard-local checkpoint saves
- rollout refresh paths
- world-size-sensitive resume
- per-stage monitoring and health checks

## Failure Modes

| Symptom | Likely cause | First check |
| ------- | ------------ | ----------- |
| hang during training | stage coordination bug | verify stage ordering and barrier alignment |
| slow training | PP bubble or overlap issue | inspect microbatch count and stage idle time |
| bad resume | stage or shard mismatch | compare checkpoint metadata with live layout |
| memory imbalance | uneven stage split | compare per-stage memory and step time |

## Review Questions

1. is PP required, or would a simpler FSDP path be easier to operate?
1. are stage count and microbatch count tuned together?
1. can every consumer of model weights understand the chosen shard layout?
1. are metrics stage-local instead of only global averages?

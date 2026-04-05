<!-- source: AReaL -->

# MoE Engine Patterns

## What This Covers

This document captures the recurring patterns behind MoE-capable engines where expert
routing and expert sharding are first-class concerns.

Use it for:

- EP and ETP layout
- router-aware engine configuration
- sparse checkpoint layout
- sparse-model onboarding and rollout refresh

## Core Principle

Sparse engines must make dense and sparse paths agree on:

- expert placement
- token dispatch and gather
- router behavior
- checkpoint naming and shard ownership

If any one of these disagrees with runtime layout, the system usually fails in ways that
look unrelated.

## Configuration Rules

1. validate `TP * PP * DP * CP * EP * ETP == world_size`
1. confirm router assumptions match expert placement
1. check that attention backend and compile mode support the sparse path
1. keep rollout or serving consumers aware of expert sharding

## Onboarding Checklist

A healthy MoE integration typically includes:

- registry entry
- config translation
- state-dict adapter
- model-specific parallelization rules
- tests for router balance and load path

## Failure Modes

| Symptom | Likely cause | First check |
| ------- | ------------ | ----------- |
| weight load mismatch | expert key layout differs from runtime | inspect state-dict adapter and expert ownership |
| poor utilization | router imbalance | log per-expert token counts and drop rate |
| slow training | token dispatch dominates | profile reorder and communication paths |
| refresh or resume failure | sparse shard ownership drift | compare expected expert-to-rank mapping |

## Review Questions

1. is the sparse shard layout explicit and stable across save, load, and refresh?
1. does the router expect the same expert grouping as runtime placement?
1. are dense-layer and expert-layer failures being separated during debugging?
1. is the sparse path verified on a reduced world-size configuration first?

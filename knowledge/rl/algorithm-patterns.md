<!-- source: AReaL -->

# RL Algorithm Patterns

## What This Covers

This document captures the recurring patterns behind PPO-family RL algorithms used for
LLM training.

Use it for:

- choosing between PPO-family variants
- debugging reward or advantage collapse
- reasoning about ratio spikes and clip behavior
- separating workflow bugs from loss bugs

## Algorithm Families

| Algorithm | Best fit | Common risk |
| --------- | -------- | ----------- |
| `PPO` | critic-based baseline with stable value bootstrap | critic bias or underfit |
| `GRPO` | grouped samples without critic | weak signal when group variance collapses |
| `Dr.GRPO` | grouped methods with unstable group std | reward scale drift |
| `GSPO` | sequence-level objectives | ratio spikes on long outputs |
| `DAPO` | PPO-style updates with dynamic packing or batching | harder effective-batch debugging |
| `RLOO` | variance reduction without critic | sensitivity to group composition |

## Stable Tuning Loop

1. confirm extraction and reward correctness
1. inspect reward distribution
1. inspect advantage distribution
1. inspect importance ratio and clipping rate
1. only then adjust learning rate, clip, or KL control

## Core Diagnostics

Useful statistics:

- reward mean and std
- advantage mean and std
- ratio mean and max
- clip rate

Interpretation:

- near-zero reward std means almost no ranking signal
- very high ratio max often means old or new logprobs are wrong
- very high clip rate often means rollout lag or step size is too large

## Reward Design Rules

- make extraction deterministic before adding clever shaping
- bound or normalize large rewards
- log raw rewards separately from processed advantages
- for grouped methods, verify that groups actually contain variance

## Review Questions

1. is this a reward bug, a workflow bug, or a loss bug?
1. do logged statistics support the proposed tuning change?
1. is the normalization scope aligned with the algorithm family?
1. is the smallest reproducible batch available for debugging?

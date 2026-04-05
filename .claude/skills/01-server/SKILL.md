---
name: remote-server-usage
description: Container-based development, GPU resource management, cluster launching, and multi-node setup for AI infrastructure
model: claude-opus-4-6
tools: [Bash, Read, Write, Agent]
---

# Remote Server Usage

This category should stay a router.

Use it when the problem is operational and remote:

- choosing or validating a remote GPU backend
- checking GPU availability and placement
- running work inside the right container
- launching or debugging multi-node infrastructure

## Included Guides

| Guide | Use For |
| ----- | ------- |
| [remote-gpu-backend.md](remote-gpu-backend.md) | local-agent plus remote-GPU execution workflows |
| [gpu-resource.md](gpu-resource.md) | GPU availability, memory estimation, and allocation |

## Quick Routing

If the task is:

- remote machine choice, container entry, code sync, remote validation ladder
  - open `remote-gpu-backend.md`
- free GPU detection, `CUDA_VISIBLE_DEVICES`, memory sizing, oversubscription avoidance
  - open `gpu-resource.md`
- launcher, Slurm, Ray, env propagation, rendezvous, resource contracts
  - open `.claude/agents/launcher-scheduler.md`
- serving topology, P/D disaggregation, KV transfer, sticky routing
  - open `.claude/agents/serving-systems-expert.md`

## Category Principles

1. keep the local agent session as control plane whenever possible
1. run GPU-dependent package management inside the target container, not on the bare host
1. check GPU availability before every GPU task
1. prefer explicit host profiles over ad hoc SSH habits
1. separate infrastructure problems from model, kernel, and serving-semantics problems

## Adjacent References

- `knowledge/distributed/launcher-scheduler-patterns.md`
- `knowledge/serving/disaggregated-serving.md`
- `knowledge/hardware/nvidia-datacenter-gpu-matrix.md`

## When Not to Use This Category

Do not start here when the real problem is:

- a CUDA crash inside one kernel
- serving-path graph breaks or compile coverage
- model registry or checkpoint onboarding
- PPO, reward, or workflow semantics

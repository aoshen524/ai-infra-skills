<!-- source: AReaL -->

# Launcher and Scheduler Patterns

## What This Covers

This document captures the reusable patterns behind cluster launchers and schedulers in
AI infrastructure systems.

Use it for:

- local, Slurm, Ray, or Kubernetes orchestration
- GPU and port allocation
- worker lifecycle and health checks
- multi-node environment propagation

## Launcher vs Scheduler

- a launcher starts processes, jobs, or pods
- a scheduler decides placement, reservations, and health management

Keep these layers separate in code and in debugging. A placement bug and a bootstrap bug
are different classes of failure.

## Effective Resource Contract

Every distributed launch should make it easy to trace:

```text
requested resources -> reserved resources -> visible devices -> used devices
```

If one stage is implicit, debugging becomes guesswork.

## Environment Propagation Chain

```text
cluster spec -> launcher wrapper -> base env -> worker override -> child runtime
```

Check this chain before touching training code when workers disagree on ports, ranks, or
visible devices.

## Operational Rules

- prefer dynamic ports over fixed ports
- prefer scheduler-derived GPU visibility over hard-coded indices
- ensure shared storage paths exist on every node
- log final launch command and effective env for at least one worker
- use cleanup paths so interrupted jobs do not leave zombie workers behind

## Failure Modes

| Symptom | Likely cause | First check |
| ------- | ------------ | ----------- |
| job never starts | broken bootstrap or env | inspect launch command and first worker log |
| no GPU visible | scheduler and runtime disagree | inspect `CUDA_VISIBLE_DEVICES` and job reservation |
| port collision | static assignment | move to reserved or dynamic ports |
| startup timeout | model init is heavier than timeout budget | raise timeout and inspect first unhealthy worker |
| multi-node rendezvous failure | service discovery or network issue | inspect address resolution and node connectivity |

## Review Questions

1. can the resource contract be traced end to end?
1. are env propagation and scheduler placement logged explicitly?
1. are startup timeouts sized for large-model initialization?
1. is there a cleanup path for failed or interrupted workers?

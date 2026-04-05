<!-- source: vLLM, AReaL -->

# Disaggregated Serving Patterns

## What This Covers

This document captures the recurring serving patterns behind prefill and decode
disaggregation.

Use it for:

- split prefill and decode deployments
- KV-cache transfer design
- sticky routing and service discovery
- topology-aware serving tradeoffs

## Request Flow

```text
router -> prefill worker -> KV transfer -> decode worker -> completion stream
```

The key serving contract is that decode resumes from transferred state instead of
repeating prefill work.

## Operational Rules

- keep request-to-decode routing sticky after assignment
- separate control-plane addressing from KV-data transfer
- size KV-transfer buffers so they do not starve normal inference
- make overflow or spill behavior explicit
- watch memory cost of P2P connectors and NCCL groups

## Failure Modes

| Symptom | Likely cause | First check |
| ------- | ------------ | ----------- |
| decode stalls after prefill | KV transfer contract is broken | inspect transfer completion and decode assignment |
| memory jumps after enabling P/D split | connector overhead | inspect extra communicator or buffer allocation |
| routing flaps between workers | sticky session contract missing | inspect router state and heartbeat drift |
| producer and consumer disagree on layout | symmetric TP assumption violated | compare shard topology across both sides |

## Review Questions

1. is the request sticky once decode begins?
1. can producer and consumer agree on KV layout and tensor parallel shape?
1. is failure handling explicit when KV transfer cannot complete?
1. are topology and memory overhead measured rather than assumed?

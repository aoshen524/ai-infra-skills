<!-- source: TensorRT-LLM -->

# Model Onboarding Debug Patterns

## What This Covers

This document captures the shortest reliable loop for debugging model onboarding and
inference-pipeline failures.

Use it for:

- model load or conversion failures
- config-translation mismatches
- weight-name or module-layout mismatches
- world-size and sharding assumptions that break startup

## Debug Loop

1. determine expected runtime shape from registry or manifest
1. run the narrowest reproducible pipeline and capture logs
1. anchor on the first causal error
1. change the smallest plausible config or mapping
1. re-run the smallest validation loop

## Common Failure Classes

- checkpoint tensor names do not match runtime parameters
- module hierarchy differs from the expected wrapper layout
- processor or tokenizer assets are wrong
- config translation missed a key such as rope or attention settings
- runtime launches with the wrong `world_size`

## Fast Iteration Rules

- use the smallest debug model or override available
- test `world_size=1` before multi-rank launch
- separate conversion, load, and generation when possible
- keep debug-only reductions labeled so they are not confused with production config

## Review Questions

1. is the registry or manifest the first thing being validated?
1. is the first specific log error identified before any patch?
1. can the issue be reproduced with a smaller model or smaller world size?
1. are weight-name and module-layout mismatches being checked early?

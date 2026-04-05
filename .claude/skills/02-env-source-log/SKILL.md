---
name: env-source-log
description: Environment, source code, and log management for AI infrastructure development. Covers build/install workflows, CI/CD patterns, and CUDA crash debugging. Use when setting up dev environments, investigating CI failures, debugging GPU crashes, or managing dependencies.
tools: [Bash, Read, Grep, Glob]
---

# Environment, Source Code & Log Management

This skill covers the operational side of AI infrastructure development: how to build, test, deploy, and debug GPU-accelerated projects.

## Sub-Topics

| File | Scope | When to Use |
|------|-------|-------------|
| [build-and-test.md](build-and-test.md) | Build, install, dependency management, linting, testing | Setting up dev environment, adding dependencies, running tests, pre-commit checks |
| [ci-workflow.md](ci-workflow.md) | CI/CD pipeline patterns, fast-fail, bot commands | Modifying CI workflows, debugging CI failures, understanding pipeline structure |
| [debug-cuda-crash.md](debug-cuda-crash.md) | CUDA crash debugging, API logging, compute-sanitizer | Debugging illegal memory access, NaN/Inf, OOM, kernel crashes |

## Cross-Cutting Principles

### Container-First Development

GPU-dependent projects should be developed inside containers to guarantee reproducible CUDA/NCCL/cuDNN versions. Never run `pip install` or `uv sync` on the bare host for projects with native GPU dependencies.

```bash
# Pattern: run commands inside the project container
docker exec -it <container> bash
# Or directly
docker exec <container> <command>
```

### Forensics Before Fixes

When diagnosing build failures, test failures, or crashes:

1. **Configuration diff** -- check config/YAML changes between working and broken states
2. **Environment diff** -- check Docker image versions, Python/CUDA versions, env vars
3. **Hardware/network topology** -- check GPU state, NCCL config, node connectivity
4. **Code changes** -- only after ruling out the above

> Source: Megatron-LM, torchtitan, FlashInfer, SGLang best practices

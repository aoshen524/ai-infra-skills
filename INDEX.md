# AI Infra Skills Index

Fastest navigation entry for Claude Code and Codex agents.

Use this file when you want the shortest route to the right expert, skill, or knowledge
document.

## Read Order

1. start here for task routing
1. open `EXPERT_KNOWLEDGE_MAP.md` if the task is domain-heavy
1. open the selected skill guide for workflow execution
1. use `CLAUDE.md` only when you need the full repo structure

## Task Routing

| Task Shape | Open First | Then |
| ---------- | ---------- | ---- |
| distributed hang, NCCL, rank mismatch | `.claude/agents/distributed-systems-expert.md` | `knowledge/distributed/*`, `.claude/skills/02-env-source-log/debug-cuda-crash.md` |
| kernel bug, SASS, TMA, micro-optimization | `.claude/agents/kernel-performance-expert.md` | `.claude/skills/05-profiler/*`, `knowledge/kernels/*` |
| serving topology, gateway, P/D, compile-region regression | `.claude/agents/serving-systems-expert.md` | `knowledge/serving/*`, `.claude/skills/05-profiler/torch-profile-triage.md` |
| model load, conversion, world-size startup crash | `.claude/agents/model-debug-agent.md` | `.claude/skills/03-design/model-onboard.md`, `knowledge/serving/model-onboarding-debug.md` |
| remote GPU validation, container, multi-machine execution | `.claude/skills/01-server/remote-gpu-backend.md` | `.claude/skills/01-server/gpu-resource.md`, `.claude/agents/launcher-scheduler.md` |
| CUDA crash, build, test, CI log reading | `.claude/skills/02-env-source-log/SKILL.md` | child guides in `02-env-source-log/` |
| rollout, reward, agent loop, online RL contract | `.claude/agents/rl-workflow-expert.md` | `knowledge/rl/*`, `.claude/skills/03-design/add-workflow.md` |
| plan review or multi-file implementation planning | `.claude/skills/04-review-plan/SKILL.md` | `review-domains.md`, `review-templates.md` |
| PR review, CI triage, commit / PR flow | `.claude/skills/06-code-review/SKILL.md` | `.claude/commands/*`, `knowledge/ci-cd/*` |

## Fast Paths

If you already know the problem family:

- distributed: `EXPERT_KNOWLEDGE_MAP.md` -> `distributed-systems-expert` row
- kernel: `.claude/skills/05-profiler/SKILL.md`
- serving: `knowledge/serving/torch-compile-coverage.md`
- hardware: `knowledge/hardware/nvidia-datacenter-gpu-matrix.md`
- CI: `knowledge/ci-cd/ci-triage-patterns.md`

## Normal Runtime Files

These are the files agents should prefer during normal task execution:

- `INDEX.md`
- `CLAUDE.md`
- `EXPERT_KNOWLEDGE_MAP.md`
- `.claude/agents/*`
- `.claude/skills/*`
- `knowledge/*`

## Maintenance-Only Files

These are useful for repo maintenance but are not primary runtime entrypoints:

- `meta/HANDOFF.md`
- `meta/REMAINING_TASKS_PROMPT.md`
- `COVERAGE_AUDIT.md`
- `SKILL_AUTHORING.md`

# SLIME Workflow Navigation

Use this guide when the target codebase is SLIME and the user needs the fastest route to
the right docs, example, or source hotspot before editing code.

## When To Use

Use it for:

- first-time orientation in the SLIME repo
- rollout, reward, or eval workflow extension
- multi-turn tool-use workflows
- choosing between Megatron and FSDP paths in SLIME
- locating the real argument or sample contract

Do not use it for:

- generic RL algorithm theory
- serving-only systems not tied to SLIME rollout code
- low-level kernel debugging

## Workflow

1. read `knowledge/rl/slime-repo-navigation.md`
1. open the closest matching SLIME example or doc first
1. confirm the public CLI surface before editing scripts
1. confirm the `Sample` contract before editing rollout or reward logic
1. only then go into backend-specific training files

## Fast Routes

- first run -> `quick_start.md`, then `usage.md`
- custom rollout or reward -> customization docs, then rollout source
- multi-turn tool use -> `examples/search-r1/`
- FSDP path -> usage docs plus FSDP example
- large-scale Megatron path -> large-model example docs plus backend code

## Use With

- `knowledge/rl/slime-repo-navigation.md`
- `knowledge/rl/workflow-contracts.md`
- `.claude/skills/03-design/add-workflow.md`
- `.claude/agents/rl-workflow-expert.md`

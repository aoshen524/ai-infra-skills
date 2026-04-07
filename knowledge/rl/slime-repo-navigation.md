# SLIME Repo Navigation

Use this document when the target framework is SLIME and the fastest path is to know which
docs, examples, and source files own a workflow concern before editing code.

## First Reads

For most SLIME tasks, start with:

1. `docs/en/get_started/quick_start.md`
1. `docs/en/get_started/usage.md`
1. one example doc or script that matches the model scale and backend

That path is usually enough to recover the basic rollout-train loop, cluster arguments,
and backend configuration without wandering the entire repo.

## Where To Look By Task

| Task | Open First | Then |
| ---- | ---------- | ---- |
| first training run | `docs/en/get_started/quick_start.md` | `docs/en/get_started/usage.md` |
| parameter meaning or CLI flags | `docs/en/get_started/usage.md` | `slime/utils/arguments.py` |
| custom reward or rollout function | `docs/en/get_started/customization.md` | `slime/rollout/sglang_rollout.py`, reward or rollout hub files |
| multi-turn tool use | `examples/search-r1/` | custom generation logic and sample contract code |
| FSDP backend setup | `docs/en/get_started/usage.md` FSDP sections | matching FSDP example |
| large-scale Megatron training | large-model example docs and scripts | backend-specific training files |
| eval dataset wiring | eval docs or config helpers | `slime/utils/eval_config.py` |

## Source Hotspots

### `slime/utils/arguments.py`

This is the fastest source file for:

- cluster-resource arguments
- rollout and data configuration
- reward-model wiring
- evaluation flags

When CLI behavior is unclear, check here before guessing from scripts alone.

### `slime/rollout/sglang_rollout.py`

Open this when the question involves:

- rollout orchestration
- async generation
- dynamic sampling
- partial rollout handling
- SGLang interaction shape

This is often the best place to understand how prompts become grouped `Sample` objects and
how generation and reward are stitched together.

### `slime/utils/types.py`

Use this file to recover the actual training-sample contract:

- `tokens`
- `response_length`
- `reward`
- `loss_mask`
- truncation or abort fields
- metadata storage

Many workflow bugs come from misunderstanding this contract.

## Example Families Worth Reusing

### Basic Single-Turn RL

Use the standard Qwen or GLM examples when you need:

- one-shot math or reasoning tasks
- a clean baseline for GRPO-style training
- a known-good script before customization

### Search-R1 and Tool Use

Use `examples/search-r1/` when you need:

- multi-turn interaction
- tool execution inside rollout
- action versus observation masking
- metadata-driven workflow state

### VLM Paths

Use the VLM examples when:

- image or multimodal data changes the sample contract
- FSDP is used for easier HF checkpoint loading
- the user needs a model-specific pattern rather than generic text-only rollout

### Async and Specialized Modes

Look at async, reproducibility, true-on-policy, or mismatch-helper examples only after a
plain synchronous path is already understood.

## Practical Reading Order

For a workflow extension:

1. read `usage.md` to recover the public configuration surface
1. read the closest matching example script
1. inspect `slime/utils/types.py` for the data contract
1. inspect `slime/rollout/sglang_rollout.py` for orchestration shape
1. only then open backend-specific training code

This keeps the public interface in view before diving into implementation details.

## Common SLIME Pitfalls

- treating training scripts as the source of truth instead of checking the real argument definitions
- editing rollout logic without re-checking the `Sample` contract
- forgetting that eval and train branches often share wiring but not exactly the same assumptions
- jumping into Megatron or FSDP backend internals before confirming the issue is not already in rollout, reward, or dataset wiring

## Relationship To Other Repo Guides

Use this document with:

- `.claude/skills/03-design/slime-workflows.md` for the task workflow
- `knowledge/rl/workflow-contracts.md` for generic rollout and reward semantics
- `knowledge/distributed/megatron-engine-patterns.md` when the issue is engine-specific
- `.claude/agents/rl-workflow-expert.md` when the problem is judgment-heavy rather than simple navigation

## Review Questions

1. are you reading the closest matching SLIME example, or a random large example?
1. is the issue in workflow wiring, reward logic, backend config, or eval config?
1. have you checked the real `Sample` contract before changing rollout code?
1. is the public CLI surface consistent with the behavior you expect from the code?

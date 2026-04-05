# RL Workflow & Pipeline Patterns

Patterns for adding new RL workflows, rollout functions, and reward functions to training frameworks.

<!-- source: AReaL (RolloutWorkflow, multi-turn, RLVR), SLIME (rollout functions, reward functions, dynamic filters) -->

## Overview

An RL training framework typically exposes three extension points for customization:

```
Workflow (orchestration)  -->  Rollout Function (data generation)  -->  Reward Function (scoring)
```

- **Workflow**: Orchestrates a single episode -- tokenize, generate, compute reward, return tensors.
- **Rollout Function**: Generates training/eval data batches, wiring engine generation and reward computation.
- **Reward Function**: Scores a single sample or group of samples.

---

## Adding a Workflow

A workflow encapsulates the logic for a single RL episode: prompt preparation, generation, reward computation, and result formatting.

<!-- source: AReaL -->

### Required Interface

```python
class MyWorkflow(RolloutWorkflow):
    async def arun_episode(
        self,
        engine: InferenceEngine,
        data: dict[str, Any],
    ) -> dict[str, torch.Tensor]:
        # 1. Prepare input_ids from data
        # 2. Build request and generate completion (async)
        # 3. Compute reward (async)
        # 4. Return result tensors
        ...
```

### Key Requirements

1. **Async and non-blocking** -- `arun_episode` must be `async def`. Never use blocking I/O (`open()` -- use `aiofiles.open()` instead).
2. **Wrap reward functions** -- use the framework's async reward wrapper (e.g., `AsyncRewardWrapper`) to make synchronous reward functions compatible.
3. **Tensor format** -- output tensors should follow the framework's convention (typically `[batch, seq_len, ...]`).
4. **Use generation helpers** -- leverage existing request/response abstractions rather than calling the engine directly.

### Multi-Turn Agent Loop Contract

<!-- source: veRL -->

For tool-using or multi-turn RL, keep the token trajectory explicit instead of rebuilding
it from final chat history.

A generic interface looks like:

```python
class AgentLoopBase(ABC):
    async def run(self, sampling_params: dict[str, Any], **kwargs) -> AgentLoopOutput:
        ...
```

Where `AgentLoopOutput` preserves:

- `prompt_ids`
- `response_ids`
- `response_mask`

This matters because:

- tool parsing may mutate message structure
- decode-encode cycles may change tokenization
- PPO-family training is sensitive to policy/trajectory mismatch

### Step-by-Step

1. **Create workflow file** -- implement a class inheriting from the base workflow API with the `arun_episode` method.
2. **Register** -- add the class to the workflow package's `__init__.py` and `__all__`.
3. **Wire to training** -- reference the workflow class path in the training script or config.
4. **Add tests** -- write async tests (`@pytest.mark.asyncio`) covering basic functionality.

### Reference Implementations

| Pattern | Description | When to Use |
|---------|-------------|-------------|
| Single-turn RLVR | One prompt, one generation, verifiable reward | Math, code, factual QA tasks |
| Multi-turn conversation | Multiple turns of dialogue with tool calling | Agent tasks, dialogue systems |
| Vision RLVR | Single-turn with image/video inputs | VLM training |

<!-- source: AReaL -->

---

## Adding a Rollout Function

A rollout function generates training or evaluation data for each rollout step, handling engine-based generation and reward computation.

<!-- source: SLIME -->

### Required Signature

```python
def generate_rollout(args, rollout_id, data_source, evaluation=False):
    if evaluation:
        return RolloutFnEvalOutput(data=eval_results)
    
    groups = data_source.get_samples(args.rollout_batch_size)
    # Generate completions, compute rewards, populate sample fields
    return RolloutFnTrainOutput(samples=groups)
```

### Key Requirements

1. **Implement both branches** -- training (`evaluation=False`) and evaluation (`evaluation=True`) must both be handled.
2. **Populate required fields** -- each training sample must have: `tokens`, `response_length`, `reward`, `status`. Include `loss_mask` if partial masking is needed.
3. **Choose the right base** -- for async RL-style generation use the engine-based rollout as a starting point; for simple SFT-style data, use file/buffer-driven rollout.
4. **Wire via CLI** -- set the function path via `--rollout-function-path module.path.generate_rollout`.

<!-- source: SLIME -->

### Evaluation Dataset Configuration

<!-- source: SLIME -->

If the rollout also powers periodic evaluation, keep eval dataset wiring explicit and
override-friendly.

Recommended pattern:

```yaml
eval:
  defaults:
    n_samples_per_eval_prompt: 1
    temperature: 0.7
    top_p: 1.0
  datasets:
    - name: aime
      path: /path/to/aime.jsonl
      rm_type: math
      input_key: prompt
      label_key: answer
      metadata_overrides:
        split: test
```

Resolution priority:

1. dataset-level overrides
1. shared eval defaults
1. CLI or global fallback values

Validation checklist:

- every dataset has a stable `name`
- input and label keys match the real file schema
- reward schema matches `reward_key` or `eval_reward_key`
- eval datasets exist whenever periodic eval is enabled

---

## Adding a Reward Function

A reward function scores generated completions to provide the training signal.

<!-- source: SLIME, AReaL -->

### Supported Modes

| Mode | Signature | When to Use |
|------|-----------|-------------|
| Single-sample | `async def custom_rm(args, sample) -> float` | Independent per-sample scoring |
| Group/batch | `async def custom_rm(args, samples) -> list[float]` | Cross-sample comparison or batch-efficient scoring |

### Key Requirements

1. **Consistent return types** -- return scalar rewards unless the pipeline explicitly supports keyed reward dicts. If using dicts, configure `reward_key` / `eval_reward_key` downstream.
2. **Explicit error handling** -- raise on invalid metadata rather than silently returning zero.
3. **Async-safe** -- avoid blocking network calls; use async HTTP clients for external reward services.
4. **Validate on edge cases** -- test behavior on truncated, failed, and empty samples.

### Optional: Reward Post-Processing

To customize normalization or shaping before advantage computation:

```python
def post_process_rewards(args, samples):
    # return (raw_rewards, processed_rewards)
    ...
```

Wire with `--custom-reward-post-process-path module.post_process_rewards`.

<!-- source: SLIME -->

---

## Testing and CI for Workflow Extensions

<!-- source: SLIME, SGLang -->

Every new workflow surface should add validation at three layers:

1. **unit tests** for parsing, schema, and reward edge cases
1. **async or integration tests** for generation + reward wiring
1. **CI registration** so the path actually executes in automation

Guidelines:

- use mocks when a real server is not needed
- choose the lightest existing CI suite that still exercises the feature
- if the repo uses generated CI workflows, edit the template and regenerate outputs
- document exact commands and hardware assumptions in the PR notes

---

## Common Mistakes

- **Forgetting the eval branch** in rollout functions -- the framework calls the same function for both training and evaluation.
<!-- source: SLIME -->
- **Blocking I/O in async workflows** -- use `aiofiles` for file operations, `await` for all async calls.
<!-- source: AReaL -->
- **Not wrapping reward functions** -- synchronous reward functions must be wrapped for async compatibility.
<!-- source: AReaL -->
- **Wrong tensor shape** -- follow the framework's conventions for output tensor dimensions.
<!-- source: AReaL -->
- **Mismatched reward schema** -- mixing scalar rewards and reward dicts without configuring the downstream `reward_key`.
<!-- source: SLIME -->
- **Missing required sample fields** -- training samples must include `tokens`, `response_length`, `reward`, and `status`.
<!-- source: SLIME -->
- **Re-encoding final chat history** in multi-turn RL -- this can drift from the actual generated token trajectory.
<!-- source: veRL -->

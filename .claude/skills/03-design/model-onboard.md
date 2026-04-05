# Model Onboarding Patterns

Patterns for adding a new model to an inference engine or serving framework, distilled from production onboarding workflows.

<!-- source: TensorRT-LLM (AutoDeploy), SGLang (diffusion pipelines) -->

## Overview

Model onboarding follows a phased, gate-checked process. Each phase produces a verifiable artifact before the next phase begins. The general flow is:

```
Survey --> Implement --> Register --> Test (hierarchical) --> Review --> E2E Run --> Ship
```

---

## Phase 0: Resource Gathering

Collect everything needed before writing code. Network fetches may require approval and the user may leave -- do all downloads upfront.

1. **Check local availability first** -- look for model code in the installed framework (e.g., `transformers`, `diffusers`) before downloading.
2. **Download code only, skip weights** -- use `--exclude "*.safetensors" "*.bin" "*.pt"` or equivalent to skip large files.
3. **GPU memory sanity check** -- estimate `num_params * bytes_per_param` and compare against available VRAM. Stop and report if insufficient.

---

## Phase 1: Survey Existing Coverage

Before writing any code, determine what already exists.

### Step 1: Check for existing implementations

- Read the model's `config.json` to identify `model_type` and `architectures`.
- Search the codebase's model directory for files that register the same config class or architecture name.
- If existing code covers the model (or a closely related variant), decide whether to **extend** or **create new**. Prefer extending if changes are minor; create new if the architecture diverges significantly. Report the decision and rationale.

### Step 2: Survey the model family

- Identify all sizes/variants in the same family (e.g., 0.6B, 8B, 32B, MoE variants).
- Determine which share the same architecture and can use a single implementation file.
- Plan to onboard the family cohesively: one model file + one test file per architecture.

### Step 3: Analyze the model architecture

Study the reference implementation (HuggingFace, official repo, or Diffusers pipeline). Identify:

- Attention type (MHA/GQA/MLA), MoE config, RoPE variant
- Normalization, activation functions
- Any data-dependent ops that may break export/compilation (e.g., `torch.nonzero`, data-conditioned branches)
- Pre-processing requirements (text encoding, latent preparation, timestep schedules for diffusion models)

<!-- source: TensorRT-LLM -->

---

## Phase 2: Implementation

### Inference Engine Models (LLM/VLM)

Write a lean, export-friendly model file:

- **Strip**: KV cache management, training paths, dropout, fallback logic, optional code paths irrelevant to the target runtime.
- **Keep**: Model hierarchy matching checkpoint structure, minimal forward signature.
- **Use canonical ops** when available (attention, RoPE, MoE, normalization). Never reimplement operations that the framework already provides optimized versions of.
- **Self-contained files**: never import from peer model implementations. Each model file is a standalone translation from the reference source.

<!-- source: TensorRT-LLM -->

### Diffusion/Generation Pipeline Models

Choose the appropriate pipeline style:

| Situation | Style |
|-----------|-------|
| Model has unique/complex pre-processing | **Monolithic pre-processing stage** -- consolidate all logic before the shared denoising loop |
| Model fits standard patterns (text-to-image, image-to-image) | **Modular composition** -- reuse existing standard stages |
| Porting a reference pipeline with many custom steps | **Monolithic** -- copy the `__call__` logic into a single stage |

Key contract: the pre-processing stage(s) must produce a batch object with all fields expected by the shared denoising/decoding stages.

<!-- source: SGLang -->

### Component Checklist

Regardless of framework, ensure:

- [ ] Model/component implementations respect the weight naming convention of the source checkpoint for automatic loading
- [ ] Configuration classes are reused from the source library when available (do not recreate)
- [ ] Parallel support (TP/SP) is considered for multi-GPU deployment

---

## Phase 3: Registration

Register the new model so the framework can discover it:

1. Register the model class with the framework's factory/registry mechanism.
2. Add import and `__all__` entry in the package `__init__.py`.
3. Add registry/config entries for all family members (different sizes may only need different resource configs).

<!-- source: TensorRT-LLM, SGLang -->

---

## Phase 3.5: Packaging and Serve Configuration

### Model Packaging Manifest

<!-- source: Ollama -->

If the runtime uses a model recipe or manifest, keep the package explicit and auditable.

Typical fields:

- base model source
- runtime parameters such as context length and stop tokens
- prompt template
- system prompt defaults
- adapter references
- minimum runtime version or compatibility requirement

Rules:

- the manifest should point to the exact base model or checkpoint source
- adapter/base-model compatibility must be explicit
- template and system defaults belong in the package recipe, not hidden in app code
- stop-token and context-window settings should be treated as part of model packaging

### Serve Config Selection

<!-- source: TensorRT-LLM -->

If onboarding includes a serve config:

1. prefer checked-in configs over hand-tuned interpolation
1. preserve the requested objective: latency, balanced, or throughput
1. keep speculative or disaggregated features disabled unless the target scenario
   explicitly requires them
1. mark interpolated knobs as needing benchmark validation

Common scenario-dependent fields:

- `max_batch_size`
- `max_num_tokens`
- `max_seq_len`
- KV cache allocation fraction
- tensor or expert parallel size
- postprocess worker count
- CUDA graph capture batch sizes

Validation checklist:

- request size fits within `max_num_tokens` after template overhead
- trust boundary is explicit when remote code or dynamic model code is enabled
- performance objective matches the selected starting config

---

## Phase 4: Hierarchical Testing

Test bottom-up. Each level must pass before proceeding to the next.

### Level 1: Block Equivalence

Test individual blocks (MLP, Attention, MoE, Norm) against the reference implementation with identical weights and inputs.

- Use actual reference classes when available (import from `transformers`, `diffusers`, etc.).
- Use RMSE-based comparison for blocks with custom ops; tight `assert_close` for identical-math blocks.
- Small config: hidden=64, layers=2-3, vocab=1000.

### Level 2: Layer Equivalence

Full decoder/transformer layer. If the model has heterogeneous layers (dense vs MoE, attention vs SSM), test each type.

### Level 3: Full Model Equivalence

End-to-end output comparison (logits, generated images/video). Use a small config with sufficient layers to cover all layer types.

### Level 4: Export/Runtime Test

Verify the model works through the framework's compilation/export path. Test with multiple input shapes.

### Numerical Tolerance Guidelines (bfloat16)

| Scope | Tolerance |
|-------|-----------|
| Identical math (MLP, Norm) | `rtol=1e-3, atol=1e-3` |
| MoE block (fused routing) | RMSE ratio < 0.02 |
| Decoder layer / full model | RMSE ratio < 0.05 |
| Attention | RMSE ratio < 0.10 |

<!-- source: TensorRT-LLM -->

---

## Phase 5: Independent Review

Have a reviewer (human or automated) independently validate the implementation:

- Provide only the model file path and test file path.
- Do not include your own assessment -- let the reviewer judge independently.
- Fix all failures and re-review until fully passing.

---

## Phase 6: End-to-End Validation

Run the model through the production serving/inference path:

1. **Reduced layers first** -- test the E2E flow with fewer layers to iterate quickly.
2. **Full model** -- run with all layers and verify output quality (coherent generation, non-noise images).

For diffusion models, common causes of noise output:
- Incorrect latent scale/shift factors
- Wrong timestep/sigma schedule
- Mismatched conditioning kwargs
- Incorrect rotary embedding style

<!-- source: SGLang -->

---

## Common Pitfalls

- **Do not repeat K/V heads manually for GQA** -- framework attention ops handle this natively.
<!-- source: TensorRT-LLM -->
- **RoPE cos/sin should be sliced once, not per layer** -- compute once and pass pre-sliced tensors.
<!-- source: TensorRT-LLM -->
- **Batch fields must use correct container types** -- e.g., prompt embeddings as list-of-tensors, sigmas as Python list not numpy.
<!-- source: SGLang -->
- **Assert required inputs** -- never include fallback logic for inputs that should always be provided (e.g., `position_ids`).
<!-- source: TensorRT-LLM -->
- **Manifest drift** -- changing runtime parameters in app code but not in the model package recipe leads to hard-to-reproduce serving bugs.
<!-- source: Ollama -->
- **Verify output quality before shipping** -- a model that produces noise or incoherent text is incorrect, even if it runs without errors.
<!-- source: SGLang -->

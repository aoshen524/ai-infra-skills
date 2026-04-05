# Rule: Model Code Conventions
<!-- sources: torchtitan -->

## Keep Models Minimal and Readable
<!-- source: torchtitan -->

- Model files should contain model architecture, not training infrastructure
- Weight initialization belongs in a config or a dedicated init function, not scattered through `Module.__init__`
- After any model change, ensure original checkpoints still load correctly

## Audit All Variants
<!-- source: torchtitan -->

- When changing shared components (attention, normalization, MoE routing), check and update **all** model variants
- Don't leave stale patterns in sibling models when updating a shared abstraction

## Unify Across Models
<!-- source: torchtitan -->

- Don't create per-model wrappers for the same functionality. Aim for at most one general wrapper shared by all models
- If multiple models have near-identical code (e.g., `apply_fsdp`, `apply_ac`, `apply_compile`), consolidate into a shared module
- Before adding a new rotary embedding, MoE router, or similar component, check if an existing implementation already supports the use case

## Standard Model Folder Structure
<!-- source: torchtitan -->

Each model folder should follow a consistent pattern:
- `config_registry.py` -- registers model configs (sizes, hyperparameters)
- `parallelize.py` -- defines parallelism strategy for the model
- Model definition files (architecture, layers)

## Shared Components
<!-- source: torchtitan -->

Components used by multiple models belong in a shared module (e.g., `models/common/`):
- Attention mechanisms
- Feed-forward layers
- Decoder blocks
- MoE routing and expert implementations
- Rotary/positional embeddings
- Normalization layers

## Don't Over-Specialize
<!-- source: torchtitan -->

- If a feature is only needed by one model, implement it in that model's folder
- Don't modify shared infrastructure or base classes to accommodate a single model's needs

## Control Flow in Forward
<!-- source: torchtitan -->

- Keep control flow (routing decisions, conditional logic) in the `forward` method
- Don't bury important branching logic inside helper methods where it's hard to trace

## Checkpoint Compatibility
<!-- source: torchtitan -->

- After any model architecture change, verify that existing checkpoints still load
- When renaming or removing parameters, provide migration logic or clear error messages

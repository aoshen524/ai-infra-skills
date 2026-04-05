# Review Templates

<!-- source: AReaL -->

Domain-specific review checklists for plan and PR review. Used by the review-plan skill to select appropriate review depth based on detected domains and signals.

---

## Template Selection Rules

1. Select templates by detected L1 domains and L2 signals
2. Use at most one primary template per domain
3. Always include **General Logic & Boundary** for non-doc/config-only changes
4. Apply cross-domain linkage checks from [review-domains.md](review-domains.md)

---

## Universal Template

### General Logic & Boundary

```
Applicable: Any non-doc/config-only change
Checklist:
- Boundary condition correctness (empty inputs, singleton, max-size)
- Conditional logic correctness (branch inversion, short-circuit mistakes)
- Error-path behavior (exceptions propagated with actionable context)
- Return-value consistency across code paths
- No newly introduced hidden behavior changes
```

---

## Domain Templates

### Domain 1: Distributed Runtime Review [comprehensive]

```
Applicable signals: parallel_dims, process_group, fsdp_core, megatron_core,
  collectives, mesh_dtensor, activation_checkpointing, weight_sync
Checklist:
- Process-group creation/usage/cleanup is rank-consistent
- Collective operations are called by all required ranks in consistent order
- DeviceMesh dimensions and DTensor placements are correct for each path
- Activation checkpoint placement remains compatible with parallel and sharding order
- Local/global tensor conversion boundaries are explicit and correct
- Weight version propagation and update ordering are deterministic
- No debug-only barriers left in hot path
```

### Domain 2: Model Compute & Attention Review [comprehensive]

```
Applicable signals: attention_impl, sequence_parallel, triton_kernel, model_family,
  moe_modeling
Checklist:
- Attention mask semantics preserved under TP/SP/CP
- Model family registration and per-family wiring remain internally consistent
- MoE router, grouped experts, and weight-conversion interfaces remain aligned
- Kernel assumptions on dtype/shape/contiguity are satisfied
- No silent behavior change in sequence packing/unpacking
- Tensor layouts remain compatible with downstream modules
```

### Domain 3: Inference Backend Review [comprehensive]

```
Applicable signals: inference_extension, remote_backend, request_lifecycle
Checklist:
- Request lifecycle (enqueue, execution, cancellation, timeout) is coherent
- Worker state transitions are safe under concurrency
- Backend-specific extension points stay API-compatible
- Error handling does not strand in-flight requests
- Weight-update interactions are explicit and safe
```

### Domain 4: Workflow & Trainer Review [comprehensive]

```
Applicable signals: workflow_engine_boundary, dataset_surface, async_contract,
  weight_version_contract
Checklist:
- Workflow and Engine interfaces remain contract-compatible
- Dataset/output structure still matches workflow and trainer consumption expectations
- Async flow uses await consistently and avoids sync I/O in async paths
- Weight update/version handshake is preserved end-to-end
- Trainer lifecycle transitions are valid for all execution branches
- Call ordering assumptions across trainer/workflow/engine are unchanged or justified
```

### Domain 5: API & Config Compatibility Review [targeted]

```
Applicable signals: dataclass_schema, cli_compat, backward_compat,
  dependency_config
Checklist:
- Public API signature and default value changes are intentional and compatible
- Dataclass validation remains complete and informative
- CLI options preserve expected compatibility semantics
- New fields include safe defaults or explicit migration handling
- Breaking changes are documented and scoped
- Dependency and build-system changes remain compatible with supported environments
```

### Domain 6: Numerics & Tensor Semantics Review [targeted]

```
Applicable signals: shape_dtype, numerical_stability, reward_surface, compile_dynamo,
  mixed_precision_fp8
Checklist:
- Tensor shape/dtype transitions are explicit and internally consistent
- Numerical stability is protected (log/division/softmax/clamp paths)
- Reward-side numerical behavior remains compatible with workflow consumption expectations
- torch.compile / dynamo assumptions still hold for dynamic shapes and distributed execution
- Mixed-precision behavior is correct for forward + backward + reduce paths
- In-place and view/reshape operations do not corrupt gradient flow
- Device placement and dtype combinations remain legal across code paths
```

### Domain 7: Checkpoint & Recovery Review [comprehensive]

```
Applicable signals: dcp_consistency, optimizer_state, resume_compat
Checklist:
- Save/load requires and enforces all-rank participation where needed
- State dict naming/structure is stable or migration-safe
- Optimizer state sharding/gather behavior is consistent
- Resume path restores model + optimizer + version state coherently
- Async checkpoint behavior preserves ordering and durability assumptions
```

### Domain 8: Launcher & Infrastructure Review [targeted]

```
Applicable signals: launcher_resource, rpc_transport, runtime_image
Checklist:
- Resource assignment matches declared parallel strategy assumptions
- RPC serialization/deserialization keeps shape/dtype/device semantics
- Transport retries/timeouts do not violate idempotency expectations
- Cross-process startup/shutdown ordering is robust
- Runtime image and build configuration remain aligned with supported variants
```

### Domain 9: CI/CD & Release Review [comprehensive]

```
Applicable signals: workflow_jobs, runner_provisioning, release_delivery
Checklist:
- Workflow triggers, job dependencies, and permissions still enforce required validation
- Runner/image provisioning remains reproducible and compatible with job expectations
- Release and deployment jobs publish the intended artifacts only
- CI changes do not silently skip tests, formatting, or release gates
```

### Domain 10: Low-Risk Hygiene Review [basic]

```
Applicable signals: tests_docs_config, logging_security
Checklist:
- Tests/docs/config edits are internally consistent and non-misleading
- Logging follows project conventions and avoids sensitive leakage
- No wildcard imports or obvious dependency hygiene regressions
- No accidental secrets/keys/tokens introduced
```

---

## Signal-Specific Add-On Checklists

Use these only when corresponding L2 signals are detected. Projects should add their own add-ons following this same pattern.

### `weight_sync` Add-On [comprehensive]

```
- Versioned updates are monotonic and race-safe
- Broadcast/all-gather points are aligned with consumer expectations
- Local caching behavior cannot serve stale weights indefinitely
```

### `activation_checkpointing` Add-On [targeted]

```
- Checkpoint wrappers are applied in a parallelism-safe order
- Selective checkpoint policies still cover the intended modules only
- Activation recompute paths do not break sharding or sequence-parallel assumptions
```

### `compile_dynamo` Add-On [targeted]

```
- torch.compile and dynamo guards still tolerate expected dynamic-shape inputs
- fullgraph and mark_dynamic choices remain compatible with distributed execution paths
- Compile-specific changes do not silently alter runtime fallback behavior
```

### `rpc_transport` Add-On [targeted]

```
- Tensor conversion is reversible and metadata-complete
- Batch fetch/request framing preserves ordering and boundaries
- Retry logic does not replay non-idempotent actions incorrectly
```

### `runtime_image_config` Add-On [targeted]

```
- Docker base image and build args still match supported backend variants
- Layer ordering preserves expected cache and dependency behavior
- Image contents remain aligned with runtime assumptions documented in the repo
```

# Review Domains and Signal Detection

<!-- source: AReaL -->

This file contains the canonical change-domain and signal detection tables for plan and PR review. Use it to classify changes by risk level and route to the appropriate review depth.

---

## Severity-to-Review-Depth Mapping

- **CRITICAL**: use `comprehensive` review depth
- **HIGH**: use `comprehensive` review depth
- **MEDIUM**: use `targeted` review depth
- **LOW**: use `basic` review depth

---

## L1 Domains and L2 Signals

### Domain 1: Distributed Runtime (CRITICAL/HIGH)

| L2 Signal | Code Pattern |
|---|---|
| `parallel_dims` | `ParallelDims`, `_build_mesh`, `apply_tp`, `apply_cp`, `ExpertTensorParallel` |
| `process_group` | `new_group`, `ProcessGroup`, `dist.get_rank(` |
| `fsdp_core` | `FSDP`, `fully_shard`, `FullyShardedDataParallel` |
| `megatron_core` | `MegatronEngine`, `pipeline`, `micro-batch` |
| `collectives` | `all_reduce`, `all_gather`, `reduce_scatter`, `all_to_all`, `broadcast`, `barrier` |
| `mesh_dtensor` | `DeviceMesh`, `DTensor`, `Shard(`, `Replicate(`, `distribute_tensor` |
| `activation_checkpointing` | `activation_checkpoint`, `checkpoint_wrapper`, `selective_checkpoint` |
| `weight_sync` | `WeightUpdateMeta`, `set_version`, `update_weights` |

### Domain 2: Model Compute & Attention (HIGH/MEDIUM)

| L2 Signal | Code Pattern |
|---|---|
| `attention_impl` | `TreeAttention`, `sdpa`, `flash_attn`, `varlen`, `causal_mask` |
| `sequence_parallel` | `SequenceParallel`, `context_parallel`, `ulysses` |
| `triton_kernel` | `triton`, `kernel`, `autotune` |
| `model_family` | `ModelSpec`, `register_model_spec`, `supported_model_types`, `rope` |
| `moe_modeling` | `TopKRouter`, `GroupedExperts`, `MoEWeightConverter`, `expert_parallel` |

### Domain 3: Inference Backend & Serving (HIGH)

| L2 Signal | Code Pattern |
|---|---|
| `inference_extension` | `vllm`, `sglang`, `server`, `worker_extension`, `pause_generation` |
| `remote_backend` | `OpenAI`, `request`, `response`, `async def generate` |
| `request_lifecycle` | `enqueue`, `dequeue`, `cancel`, `timeout` |

### Domain 4: Workflow & Trainer Contract (HIGH/MEDIUM)

| L2 Signal | Code Pattern |
|---|---|
| `workflow_engine_boundary` | `RolloutWorkflow`, `arun_episode`, `agenerate` |
| `dataset_surface` | `DataLoader`, `IterableDataset`, `get_*_dataset` |
| `async_contract` | `async def`, `await`, `aiofiles`, `asyncio` |
| `weight_version_contract` | `WeightUpdateMeta`, `set_version`, `weight version` |

### Domain 5: API & Config Compatibility (MEDIUM)

| L2 Signal | Code Pattern |
|---|---|
| `dataclass_schema` | `@dataclass`, `field(`, `__post_init__` |
| `cli_compat` | `Literal`, `help`, `default` |
| `backward_compat` | `deprecated`, `compat`, `version` |
| `dependency_config` | `requires-python`, `dependencies`, `optional-dependencies` |

### Domain 6: Numerics & Tensor Semantics (MEDIUM)

| L2 Signal | Code Pattern |
|---|---|
| `shape_dtype` | `.view(`, `.reshape(`, `dtype=`, `.contiguous(` |
| `numerical_stability` | `log(`, `softmax`, `eps=`, `.clamp(`, `nan`, `inf` |
| `reward_surface` | `reward_fn`, `AsyncRewardWrapper` |
| `compile_dynamo` | `torch.compile`, `_dynamo`, `mark_dynamic`, `fullgraph` |
| `mixed_precision_fp8` | `fp8`, `bf16`, `fp16`, `mixed precision` |

### Domain 7: Checkpoint & Recovery (CRITICAL/HIGH)

| L2 Signal | Code Pattern |
|---|---|
| `dcp_consistency` | `dcp.save`, `dcp.load`, `DistributedCheckpoint` |
| `optimizer_state` | `optimizer state`, `state_dict` |
| `resume_compat` | `resume`, `load_state_dict`, `migration` |

### Domain 8: Launcher & Infrastructure (HIGH/MEDIUM)

| L2 Signal | Code Pattern |
|---|---|
| `launcher_resource` | `LaunchConfig`, `RayLauncher`, `SlurmLauncher` |
| `rpc_transport` | `RTensor`, `serialize`, `rpc`, `fetch` |
| `runtime_image` | `FROM`, `ARG`, `RUN`, `ENV`, `COPY` |

### Domain 9: CI/CD & Release (HIGH/CRITICAL)

| L2 Signal | Code Pattern |
|---|---|
| `workflow_jobs` | `jobs:`, `runs-on:`, `needs:`, `if:`, `workflow_dispatch` |
| `runner_provisioning` | `runner`, `image`, `heartbeat` |
| `release_delivery` | `docker`, `tag`, `release`, `pages`, `publish` |

### Domain 10: Low-Risk Hygiene (LOW)

| L2 Signal | Code Pattern |
|---|---|
| `tests_docs_config` | `tests/`, `docs/`, `*.md`, `*.yaml`, `*.json` |
| `logging_security` | `getLogger`, `print(`, `import *`, `api_key`, `token`, `password` |

---

## Cross-Domain Linkage Rules

When one domain is detected, automatically include checks from linked domains:

| Detected Signal | Auto-Linked Review |
|---|---|
| `parallel_dims` or `process_group` | Model Compute & Attention checks |
| `model_family` or `moe_modeling` | Numerics & Tensor Semantics checks |
| `attention_impl` | Numerics & Tensor Semantics checks |
| `reward_surface` | Workflow & Trainer Contract checks |
| `compile_dynamo` | Distributed Runtime checks |
| `inference_extension` | Launcher & Infrastructure checks |
| `weight_sync` | DTensor/process-group/checkpoint interaction checks |
| `rpc_transport` | Distributed Runtime synchronization checks |
| `mixed_precision_fp8` + Distributed Runtime | mesh + weight-sync compatibility checks |
| `runtime_image` | Inference Backend & Serving checks |
| `dependency_config` | API & Config Compatibility checks |
| `workflow_jobs` or `release_delivery` | Launcher & Infrastructure checks |

---

## Risk Identification by Domain

### Distributed Runtime Risks
- Mesh construction or parallel-dims mismatch
- EP/TP/CP application order errors
- Activation checkpoint placement violating parallel ordering assumptions
- Collective call order mismatch across ranks
- Wrong process-group scope in rank-sensitive logic
- Weight version drift between rollout and training workers

### Model Compute & Attention Risks
- Attention mask inconsistency under TP/SP/CP paths
- Model-family registration or per-family wiring drift
- MoE router/expert behavior diverging from weight-conversion expectations
- Kernel assumptions violating dtype/shape invariants
- Sequence packing alignment errors

### Inference Backend & Serving Risks
- Request lifecycle inconsistencies (enqueue/cancel/timeout)
- Worker state transitions leaving requests stranded
- Backend extension hooks drifting from runtime expectations

### Workflow & Trainer Contract Risks
- Workflow-engine contract drift across async boundaries
- Weight version handshake mismatch between rollout and train
- Trainer lifecycle transition inconsistencies

### API & Config Compatibility Risks
- Breaking config/schema changes without migration path
- Dataclass or CLI default changes altering behavior silently
- Dependency pin changes breaking supported environments

### Numerics & Tensor Semantics Risks
- Silent shape/dtype mismatch under distributed paths
- Unstable numerical operations in loss/reward logic
- torch.compile guard changes breaking graph assumptions
- Mixed-precision interaction regressions

### Checkpoint & Recovery Risks
- Partial-rank checkpoint participation
- Incompatible state key evolution
- Resume path breaking optimizer/model synchronization

### Launcher & Infrastructure Risks
- Resource assignment mismatching parallel strategy assumptions
- RPC transport metadata loss (shape/dtype/device)
- Runtime image drift from supported variants

### CI/CD & Release Risks
- Workflow trigger changes skipping required validation
- Runner provisioning drift causing flaky CI
- Release jobs publishing wrong artifacts

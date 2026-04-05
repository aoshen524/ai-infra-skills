# Engine Architecture & RL Algorithm Patterns

Patterns for choosing, configuring, and extending distributed training engines and RL algorithms in LLM training frameworks.

<!-- source: AReaL (FSDPEngine, ArchonEngine, MegatronEngine, GRPO/PPO/DAPO algorithms) -->

## Overview

A distributed RL training framework typically provides multiple engine backends and multiple RL algorithm implementations. Choosing the right combination depends on model architecture, cluster topology, and training objective.

```
Engine (distributed training)  -->  Algorithm (loss computation)  -->  Workflow (rollout orchestration)
```

---

## Engine Selection

| Engine Type | Best For | Key Features |
|-------------|----------|--------------|
| **FSDP-based** | Dense transformer models | FSDP2 parameter sharding, TP/DP/CP parallelism, CPU offloading |
| **MoE-specialized** | Mixture-of-Experts models | Expert-parallel sharding, load balancing, MoE-specific communication |
| **Pipeline-parallel** | Ultra-deep models exceeding single-node memory | Pipeline stages across nodes, micro-batch scheduling |

<!-- source: AReaL -->

---

## FSDP Engine Configuration

The FSDP engine is the general-purpose choice for dense models. Configuration combines training settings with parallel strategy.

### Configuration Components

| Component | Purpose |
|-----------|---------|
| **TrainEngineConfig** | Core training parameters: optimization, checkpointing, data types |
| **ParallelStrategy** | Parallel dimensions: tensor (TP), data (DP), context (CP) parallelism sizes |
| **FSDPEngineConfig** | FSDP-specific: wrap policy, CPU offloading, memory-efficient loading |

### Parallel Strategy Guidelines

- **Memory-constrained**: prefer DP over TP, enable CPU offloading and memory-efficient loading.
- **Performance-optimized**: balance TP/DP/CP based on model width and cluster bandwidth.
- **Scaling rules**: increase DP for larger batches, TP for wider models, CP for longer sequences.
- **Dimension constraint**: `dp * sp * tp == world_size` must always hold.

### Algorithm-Specific Subclasses

FSDP engines provide algorithm-specific subclasses with integrated training logic:

| Subclass | Use Case |
|----------|----------|
| PPO Actor / Critic | PPO reinforcement learning with separate actor and critic models |
| LM Engine | Supervised fine-tuning (SFT) |
| Reward Model Engine | Preference modeling and reward model training |

### Weight Synchronization

Two mechanisms for updating rollout engines from training engines:

| Mechanism | When to Use |
|-----------|-------------|
| **XCCL (NCCL)** | Low-latency broadcast for homogeneous GPU clusters |
| **Disk-based** | File exchange for heterogeneous clusters or fault tolerance |

<!-- source: AReaL -->

---

## RL Algorithm Family

PPO-family algorithms differ primarily in normalization, clipping, and baseline strategies.

<!-- source: AReaL -->

### Algorithm Comparison

| Algorithm | Key Features | Typical Config Override |
|-----------|-------------|----------------------|
| **PPO** | Critic-based, GAE advantage estimation | `kl_ctl > 0` |
| **GRPO** | Critic-free, group normalization | Default config |
| **Dr.GRPO** | Mean-only normalization (no std) | `adv_norm.std_level = null` |
| **DAPO** | Dynamic batch sizing | Dedicated config file |
| **RLOO** | Leave-one-out baseline | Dedicated config file |
| **GSPO** | Sequence-level importance sampling | `importance_sampling_level = sequence` |

### Core Parameters

```
eps_clip      = 0.2    # PPO clipping parameter
kl_ctl        = 0.1    # KL penalty coefficient (0 for critic-free methods)
discount      = 1.0    # gamma for future rewards
gae_lambda    = 1.0    # GAE lambda parameter
```

### Advantage Normalization

| Level | Behavior |
|-------|----------|
| `batch` | Normalize across all samples in the batch |
| `group` | Normalize within each prompt group (GRPO default) |
| `None` | No normalization |

### PPO Loss Formula

```
L = -min(r(theta) * A, clip(r(theta), 1-eps, 1+eps) * A)

where:
  r(theta) = pi_new / pi_old  (importance ratio)
  A = advantage (normalized per config)
  eps = eps_clip
```

---

## Common Usage Patterns

| Scenario | Recommended Setup |
|----------|-------------------|
| **Memory-constrained training** | High DP, CPU offloading enabled, memory-efficient loading |
| **High-throughput training** | Balanced TP/DP/CP, NCCL weight updates |
| **PPO with critic** | Separate actor/critic engine instances, different offloading strategies |
| **Critic-free RL (GRPO/DAPO)** | Single actor engine, group-level advantage normalization |
| **LoRA fine-tuning** | Data parallelism with base model offloading |

---

## Troubleshooting

| Symptom | Likely Cause | First Steps |
|---------|-------------|-------------|
| Initialization failure | Invalid parallel dimensions | Verify `dp * sp * tp == world_size` |
| Out of memory | Insufficient GPU memory | Enable `offload_params=True`, reduce batch size |
| Poor throughput | Suboptimal parallel strategy | Profile and adjust TP/DP/CP balance |
| Weight sync failure | Network/NCCL issues | Switch to disk-based updates, check network |
| KL divergence explodes | Clipping too loose | Reduce `eps_clip`, increase `kl_ctl` |
| No learning signal | Advantage normalization issue | Check `adv_norm` config, ensure reward variance exists |
| Reward always 0/1 | Reward function bug | Check extraction logic in reward function |

### Debugging RL Training

```python
# Check advantage statistics
print(f"Advantages: mean={adv.mean():.4f}, std={adv.std():.4f}")
print(f"Rewards: mean={rewards.mean():.4f}, std={rewards.std():.4f}")

# Check importance ratio
ratio = (new_logp - old_logp).exp()
print(f"Importance ratio: mean={ratio.mean():.4f}, max={ratio.max():.4f}")

# Check clipping frequency
clipped = (ratio < 1 - eps) | (ratio > 1 + eps)
print(f"Clipping rate: {clipped.float().mean():.2%}")
```

<!-- source: AReaL -->

---

## Implementation Structure

A typical FSDP engine implementation includes:

| Component | Purpose |
|-----------|---------|
| **Core engine** | Main engine class and algorithm-specific subclasses |
| **Parallel helpers** | Mesh construction, dimension validation, TP application |
| **FSDP wrapping** | FSDP2 module wrapping with mixed precision and offload policies |
| **Checkpointing** | Distributed checkpoint (DCP) save/load |
| **Gradient handling** | TP/DP/PP-aware gradient norm clipping |
| **Optimizer** | Mixed-precision optimizer (e.g., AnyPrecisionAdamW with Kahan summation) |
| **Weight updates** | XCCL broadcast or disk-based exchange for rollout engine sync |
| **Model patches** | VLM-specific TP adaptations (e.g., vision tower parallelism) |
| **Sequence parallelism** | Ulysses SP communication primitives and attention patches |

<!-- source: AReaL -->

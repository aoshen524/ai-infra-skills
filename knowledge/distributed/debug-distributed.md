<!-- source: AReaL -->

# Debugging Distributed Training Issues (FSDP2, TP, CP, EP)

## Minimal Reproduction Principle

Before diving into distributed debugging, always try to reproduce with the smallest possible configuration:

1. Reduce to 2 GPUs (minimum for distributed)
2. Use a tiny model (e.g., 125M params)
3. Use synthetic data (eliminates data pipeline issues)
4. Disable mixed precision initially (eliminates dtype issues)
5. Disable gradient checkpointing (eliminates recomputation bugs)

If the issue reproduces at small scale, debugging is 10x faster. If it only reproduces at scale, that itself is a clue (usually memory or communication timeout).

## Hang Debugging

### Essential Environment Variables

| Variable | Value | Purpose |
|---|---|---|
| `TORCH_DISTRIBUTED_DEBUG` | `DETAIL` | Logs all collective calls with shapes and process groups |
| `NCCL_DEBUG` | `INFO` | NCCL-level logging (connection, ring topology) |
| `NCCL_DEBUG_SUBSYS` | `ALL` or `COLL,NET` | Filter NCCL subsystem logs |
| `TORCH_SHOW_CPP_STACKTRACES` | `1` | Show C++ stack on crash |
| `CUDA_LAUNCH_BLOCKING` | `1` | Synchronous CUDA execution (pinpoints exact failing op) |
| `NCCL_ASYNC_ERROR_HANDLING` | `1` | Detect NCCL errors without full hang |
| `NCCL_BLOCKING_WAIT` | `0` | Non-blocking NCCL wait (combine with ASYNC_ERROR) |

### Using py-spy for Stuck Processes

When a training job hangs, `py-spy` reveals where each rank is stuck:

```bash
# Find the PID for each rank
ps aux | grep python

# Dump stack trace for a specific rank
py-spy dump --pid <PID>

# Record a flame graph (attach to running process)
py-spy record --pid <PID> -o profile_rank0.svg --duration 10
```

Common patterns in py-spy output:
- All ranks stuck in `ncclAllReduce` / `ncclReduceScatter`: one rank hasn't reached the collective
- One rank in `backward()`, others in `all_reduce`: gradient computation imbalance
- All ranks in `barrier()`: someone called barrier with wrong process group

### Common Hang Causes

1. **Mismatched collectives**: Rank 0 calls `all_reduce` on param A, Rank 1 calls `all_reduce` on param B. Happens when model structure differs across ranks (conditional layers).
2. **Wrong process group**: Using `WORLD` group when you should use the TP/DP sub-group, or vice versa. The collective completes on a subset of ranks while others wait on the wrong group.
3. **Shape mismatch in all_gather**: FSDP2 all-gathers sharded params. If one rank has a different shard shape (e.g., due to uneven vocab size padding), the collective hangs or crashes.
4. **Deadlock in pipeline parallelism**: Send/recv ordering mismatch between pipeline stages.
5. **CUDA OOM on one rank**: One rank OOMs silently while others wait for its collective contribution.

## Wrong Results Debugging

### Check DTensor Placements

With FSDP2 and TP, parameters are DTensors with specific placements (Shard, Replicate). Wrong placements cause silent correctness issues:

```python
for name, param in model.named_parameters():
    if hasattr(param, 'placements'):
        print(f"{name}: shape={param.shape}, placements={param.placements}")
```

Common issues:
- Parameter should be Shard(0) but is Replicate (no sharding, wasteful but "works")
- Parameter should be Replicate but is Shard (each rank sees partial data, wrong gradients)

### Verify Gradient Reduction

After backward pass, check that gradients are consistent across ranks:

```python
for name, param in model.named_parameters():
    if param.grad is not None:
        grad_sum = param.grad.sum()
        # All ranks should see the same value after reduction
        dist.all_reduce(grad_sum, op=dist.ReduceOp.SUM)
        if rank == 0:
            print(f"{name}: grad_sum={grad_sum.item()}")
```

### Rank-Conditional Printing

```python
def rank_print(msg, target_rank=0):
    if dist.get_rank() == target_rank:
        print(f"[Rank {target_rank}] {msg}")
```

## OOM Debugging

### Check Memory Allocation

```python
import torch

def print_memory_stats(prefix=""):
    rank = dist.get_rank()
    allocated = torch.cuda.memory_allocated() / 1e9
    reserved = torch.cuda.memory_reserved() / 1e9
    max_allocated = torch.cuda.max_memory_allocated() / 1e9
    print(f"[Rank {rank}] {prefix} "
          f"allocated={allocated:.2f}GB reserved={reserved:.2f}GB "
          f"max_allocated={max_allocated:.2f}GB")
```

### Verify FSDP Coverage

Every parameter should be managed by FSDP. Un-wrapped parameters remain fully materialized on every rank:

```python
for name, param in model.named_parameters():
    is_dtensor = hasattr(param, '_local_tensor')
    if not is_dtensor:
        print(f"WARNING: {name} is NOT an FSDP DTensor "
              f"(shape={param.shape}, {param.numel() * param.element_size() / 1e6:.1f}MB)")
```

## Communication Errors Table

| Error Message | Likely Cause | Fix |
|---|---|---|
| `NCCL WARN Cuda failure 'peer access is not supported'` | GPU topology doesn't support P2P | Set `NCCL_P2P_DISABLE=1` or check PCIe topology |
| `NCCL WARN Timeout on communicator` | Rank didn't reach collective in time | Check for hangs, increase `NCCL_TIMEOUT` |
| `RuntimeError: Invalid device mesh` | Process group creation failed | Check WORLD_SIZE, RANK, MASTER_ADDR/PORT |
| `NCCL WARN Bootstrap: no socket interface found` | Network interface issue | Set `NCCL_SOCKET_IFNAME=eth0` (or correct interface) |
| `Watchdog caught collective operation timeout` | Collective timeout (default 30min) | Debug the hang; don't just increase timeout |
| `NCCL WARN Call to ibv_modify_qp failed` | InfiniBand issue | Check IB status with `ibstat`, verify RDMA config |

## Tensor Consistency Validation

For verifying that a distributed implementation matches single-GPU reference:

```python
def validate_tensor_across_ranks(tensor, name="tensor"):
    """Check that a tensor is identical across all ranks."""
    world_size = dist.get_world_size()
    gathered = [torch.zeros_like(tensor) for _ in range(world_size)]
    dist.all_gather(gathered, tensor)

    for i in range(1, world_size):
        if not torch.equal(gathered[0], gathered[i]):
            diff = (gathered[0] - gathered[i]).abs()
            print(f"MISMATCH {name}: rank 0 vs rank {i}, "
                  f"max_diff={diff.max().item()}, "
                  f"mean_diff={diff.mean().item()}")
            return False
    return True
```

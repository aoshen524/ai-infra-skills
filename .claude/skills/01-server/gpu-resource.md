---
name: gpu-resource-management
description: GPU availability checking, memory estimation, multi-GPU allocation strategies, and common GPU failure remediation
model: claude-opus-4-6
tools: [Bash, Read, Write, Agent]
---

# GPU Resource Management

Detailed guide for GPU resource checking, memory estimation, allocation strategies, and failure remediation in AI training infrastructure.

---

## 1. Checking GPU Availability

### Before Every GPU Task

**Rule**: Never launch a GPU task without checking availability first.

<!-- source: verl-grounding, TensorRT-LLM -->

```bash
# Check if any processes are using GPUs (empty output = all free)
nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv,noheader

# Detailed per-GPU status
nvidia-smi --query-gpu=index,memory.used,memory.total,utilization.gpu --format=csv,noheader,nounits
```

### Inside a Docker Container

```bash
# From the host, query GPU status inside a container
docker exec <container> nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv,noheader

# Or from inside the container directly
nvidia-smi --query-gpu=index,memory.used,utilization.gpu --format=csv,noheader,nounits
```

<!-- source: verl-grounding -->

### Interpreting Results

A GPU is considered **free** when:
- Memory usage < ~1000 MiB (small residual from driver/display)
- GPU utilization is 0%

A GPU is considered **busy** when:
- Any compute process is listed, OR
- Memory usage > 1000 MiB, OR
- GPU utilization > 0%

### Automated Availability Check Pattern

```bash
# One-liner: list free GPU indices (memory < 1000 MiB, util == 0%)
nvidia-smi --query-gpu=index,memory.used,utilization.gpu \
  --format=csv,noheader,nounits | \
  awk -F', ' '$2 < 1000 && $3 == 0 {print $1}'
```

---

## 2. GPU Memory Estimation

### Rule of Thumb: Model Parameters to Memory

| Component | Memory per Parameter (fp16/bf16) | Memory per Parameter (fp32) |
|-----------|----------------------------------|----------------------------|
| Model weights | 2 bytes | 4 bytes |
| Gradients | 2 bytes | 4 bytes |
| Optimizer states (Adam) | 8 bytes (master weights + m + v) | 12 bytes |
| Activations | Varies (batch-dependent) | Varies |

### Quick Estimation Formula

```
Training memory ~= params * 18 bytes (fp16 + Adam optimizer)
                 + activation memory (batch_size dependent)
                 + CUDA kernel overhead (~500 MiB - 1 GiB)

Inference memory ~= params * 2 bytes (fp16/bf16)
                  + KV cache (batch * seq_len * layers * hidden * 4 bytes)
                  + overhead (~500 MiB)
```

### Examples

| Model Size | Inference (fp16) | Training (fp16 + Adam) |
|-----------|-----------------|----------------------|
| 1.5B | ~3 GiB | ~30 GiB |
| 7B | ~14 GiB | ~140 GiB |
| 13B | ~26 GiB | ~260 GiB |
| 70B | ~140 GiB | ~1.4 TiB |

### Memory Reduction Techniques

| Technique | Memory Savings | Trade-off |
|-----------|---------------|-----------|
| Gradient checkpointing | ~60% activation memory | ~30% slower |
| ZeRO Stage 1 (optimizer sharding) | Optimizer states / N GPUs | Minimal overhead |
| ZeRO Stage 2 (+ gradient sharding) | (Optimizer + gradients) / N | Small comm overhead |
| ZeRO Stage 3 / FSDP (+ param sharding) | All / N GPUs | Higher comm overhead |
| LoRA / QLoRA | Train only ~1% of params | Reduced expressiveness |
| Mixed precision (bf16) | ~50% vs fp32 | Minimal quality loss |
| Flash Attention | O(N) vs O(N^2) memory | Requires compatible GPU |

---

## 3. Multi-GPU Allocation Strategies

### Single-Node Multi-GPU

<!-- source: TensorRT-LLM -->

**Step 1 -- Determine required GPU count** from model config:

```bash
# Common config fields: world_size, num_gpus, tensor_parallel_size
grep -r "world_size\|num_gpus\|tensor_parallel" config.yaml
```

**Step 2 -- Select free GPUs** (prefer contiguous, lowest indices):

```bash
# Query available GPUs
nvidia-smi --query-gpu=index,memory.used,utilization.gpu --format=csv,noheader,nounits

# Set visibility for selected GPUs
export CUDA_VISIBLE_DEVICES=0,1,2,3
```

**Step 3 -- Launch with explicit GPU binding**:

```bash
# torchrun (PyTorch distributed)
CUDA_VISIBLE_DEVICES=0,1,2,3 torchrun --nproc_per_node=4 train.py

# Single-GPU task on a specific device
CUDA_VISIBLE_DEVICES=2 python inference.py
```

### Multi-Node Multi-GPU

<!-- source: AReaL -->

```bash
# Node 0 (head)
MASTER_ADDR=node0 MASTER_PORT=29500 \
  torchrun --nnodes=2 --nproc_per_node=8 --node_rank=0 \
  --master_addr=node0 --master_port=29500 train.py

# Node 1 (worker)
MASTER_ADDR=node0 MASTER_PORT=29500 \
  torchrun --nnodes=2 --nproc_per_node=8 --node_rank=1 \
  --master_addr=node0 --master_port=29500 train.py
```

### GPU Topology Awareness

For optimal performance, consider GPU interconnect topology:

```bash
# Check GPU topology (NVLink, PCIe, etc.)
nvidia-smi topo -m

# Prefer GPUs connected via NVLink for tensor parallelism
# Use PCIe-connected GPUs across pipeline stages
```

### Allocation Decision Tree

```
Need 1 GPU?
  -> Pick the free GPU with lowest index
  -> CUDA_VISIBLE_DEVICES=<index>

Need N GPUs on 1 node?
  -> Pick N contiguous free GPUs (lowest indices)
  -> CUDA_VISIBLE_DEVICES=<idx1>,<idx2>,...
  -> torchrun --nproc_per_node=N

Need GPUs across nodes?
  -> Verify all nodes have enough free GPUs
  -> Set MASTER_ADDR, MASTER_PORT, WORLD_SIZE
  -> Launch with torchrun --nnodes=M --nproc_per_node=N

Not enough free GPUs?
  -> Report busy GPUs (show PID, process, memory)
  -> Wait and re-check, do NOT oversubscribe
```

<!-- source: TensorRT-LLM -->

---

## 4. Common GPU Failures and Fixes

### Out of Memory (OOM)

```
RuntimeError: CUDA out of memory. Tried to allocate X GiB
```

**Diagnosis**:
```bash
# Check what is consuming memory
nvidia-smi
# Check for memory leaks (growing memory over steps)
watch -n 1 nvidia-smi
```

**Fixes** (in order of preference):
1. Reduce `per_device_batch_size`
2. Enable gradient checkpointing (`gradient_checkpointing=True`)
3. Use mixed precision (`bf16=True`)
4. Enable FSDP / ZeRO sharding
5. Reduce `max_seq_length`
6. Use fewer GPUs per model replica, more replicas with smaller batch

### ECC Errors

```
Xid 48: ECC error detected
```

**Diagnosis**:
```bash
# Check for ECC errors
nvidia-smi --query-gpu=index,ecc.errors.corrected.volatile.total,ecc.errors.uncorrected.volatile.total --format=csv
```

**Fix**: Uncorrectable ECC errors require GPU reset or node replacement. Report to cluster admin.

### NCCL Communication Failures

```
NCCL error: unhandled system error / NCCL timeout
```

**Diagnosis**:
```bash
# Enable NCCL debug logging
export NCCL_DEBUG=INFO
export NCCL_DEBUG_SUBSYS=ALL

# Check network interface
ip addr show
export NCCL_SOCKET_IFNAME=eth0  # or appropriate interface
```

**Fixes**:
1. Verify all nodes can reach each other on the NCCL port
2. Set `NCCL_SOCKET_IFNAME` to the correct network interface
3. Increase timeout: `NCCL_TIMEOUT=1800`
4. For InfiniBand: check `ibstat` and `NCCL_IB_DISABLE=0`

### GPU Process Stuck / Zombie

```bash
# Find stuck GPU processes
nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv,noheader

# Kill stuck processes
kill -9 <pid>

# Nuclear option: kill all GPU processes (use with caution)
nvidia-smi --query-compute-apps=pid --format=csv,noheader | xargs -r kill -9

# If GPU state is corrupted, reset the GPU (requires root)
nvidia-smi --gpu-reset -i <gpu_index>
```

### CUDA Version Mismatch

```
RuntimeError: The NVIDIA driver on your system is too old
```

**Diagnosis**:
```bash
# Check driver version
nvidia-smi | head -3

# Check CUDA runtime version
nvcc --version

# Check PyTorch CUDA version
python -c "import torch; print(torch.version.cuda)"
```

**Fix**: Use a container with matching CUDA toolkit version. Do not try to upgrade the host driver.

### GPU Not Visible

```
RuntimeError: No CUDA GPUs are available
```

**Diagnosis**:
```bash
# Check if GPUs are visible to the system
nvidia-smi

# Check CUDA_VISIBLE_DEVICES is not accidentally empty
echo $CUDA_VISIBLE_DEVICES

# Inside Docker: verify --gpus flag was passed
docker inspect <container> | grep -A5 DeviceRequests
```

**Fix**:
- Ensure container is started with `--gpus all` or `--gpus '"device=0,1"'`
- Check that `CUDA_VISIBLE_DEVICES` is not set to empty string or invalid indices
- Verify the NVIDIA Container Toolkit is installed on the host

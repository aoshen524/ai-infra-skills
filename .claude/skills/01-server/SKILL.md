---
name: remote-server-usage
description: Container-based development, GPU resource management, cluster launching, and multi-node setup for AI infrastructure
model: claude-opus-4-6
tools: [Bash, Read, Write, Agent]
---

# Remote Server Usage

This skill covers the essential patterns for working on remote GPU servers in AI
infrastructure projects: remote GPU backends, container-based development, GPU resource
management, cluster launching, and session management.

Included guides:

- `remote-gpu-backend.md` for using local agent sessions with remote GPU execution hosts
- `gpu-resource.md` for GPU availability and allocation reasoning

---

## 1. Container-Based Development

### Why Containers

AI training depends on CUDA, NCCL, cuDNN, PyTorch with GPU support, and various native extensions. Installing these on a bare host is fragile and hard to reproduce. **Use the container as your development environment** to guarantee:

- Identical CUDA / NCCL / cuDNN versions across all developers and CI
- Dependency lock files resolve the same way locally and in CI
- GPU-dependent operations work out of the box

<!-- source: Megatron-LM -->

### Running Commands Inside a Container

```bash
# Enter an existing container interactively
docker exec -it <container_name> bash

# Run a one-off command in a container
docker exec <container_name> <command>

# Launch a new container with GPU access and workspace mount
docker run --rm --gpus all \
  -v $(pwd):/workspace \
  -w /workspace \
  <image>:<tag> \
  bash -c "<your command>"
```

<!-- source: Megatron-LM -->

### Image Management

```bash
# Pull a pre-built image
docker pull <registry>/<image>:<tag>

# Build from a Dockerfile
docker build \
  --build-arg FROM_IMAGE_NAME=<base_image> \
  -f docker/Dockerfile \
  -t <image>:local .

# List running containers
docker ps

# List all images
docker images
```

### Key Principle

> **All dependency management (pip, uv, conda) must be run inside the container.** Never install GPU-dependent packages on the host.

<!-- source: Megatron-LM -->

| Problem | Cause | Fix |
|---------|-------|-----|
| `ModuleNotFoundError` after pip install | Installed outside the container venv | Use the container's package manager, never bare `pip install` on host |
| `No space left on device` during installs | Cache fills container storage | Mount a host cache dir via `-v $HOME/.cache:/root/.cache` |
| Port collision on multi-GPU runs | torchrun binding conflicts | Use `torch.distributed.run` via the container entry point |

<!-- source: Megatron-LM -->

---

## 2. GPU Resource Management

### Pre-Flight Check (Mandatory)

Before running any GPU command, **always check GPU availability first**:

```bash
# Quick check: are any GPUs occupied?
nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv,noheader

# Detailed per-GPU status: index, memory used, utilization
nvidia-smi --query-gpu=index,memory.used,utilization.gpu --format=csv,noheader,nounits
```

<!-- source: TensorRT-LLM, verl-grounding -->

**Decision rules**:
- If output is empty (no processes) -> GPUs are free, proceed
- If processes are running -> report which GPUs are busy, do NOT launch GPU tasks
- When running inside a container: `docker exec <container> nvidia-smi ...`

### GPU Selection Strategy

<!-- source: TensorRT-LLM -->

1. **Determine required GPU count** from model config (e.g., `world_size` field)
2. **Query availability**: a GPU is "free" if memory < ~1000 MiB and utilization is 0%
3. **Select GPUs**: pick N contiguous free GPUs (prefer lowest indices)
4. **Set visibility**: `CUDA_VISIBLE_DEVICES=<comma-separated indices>`

```bash
# Example: select 2 free GPUs for a job
CUDA_VISIBLE_DEVICES=0,1 python train.py
```

If not enough GPUs are available, **wait and retry** rather than oversubscribing:

```bash
# Check again after waiting
nvidia-smi --query-gpu=index,memory.used,utilization.gpu --format=csv,noheader,nounits
```

See [gpu-resource.md](gpu-resource.md) for detailed GPU memory estimation and allocation patterns.

---

## 3. Multi-Node Cluster Setup

### Launcher vs. Scheduler

<!-- source: AReaL -->

| Component | Responsibility | Examples |
|-----------|---------------|----------|
| **Launcher** | Starts processes, manages process tree, passes environment variables | Local, Slurm, Ray launchers |
| **Scheduler** | Allocates GPU/port resources, manages worker lifecycle, health checks | Local, Slurm, Ray schedulers |

### Ray Cluster

```bash
# Start Ray head node
ray start --head --port=6379 --num-gpus=<N>

# Join worker nodes to the cluster
ray start --address=<head_ip>:6379 --num-gpus=<N>

# Check cluster status
ray status

# Stop Ray on current node
ray stop
```

### Slurm

```bash
# Submit a training job
sbatch --nodes=<N> --gres=gpu:<G> --partition=<queue> train.sbatch

# Check job status
squeue -u $USER

# Cancel a job
scancel <job_id>

# Interactive GPU session
srun --nodes=1 --gres=gpu:8 --pty bash
```

### Environment Variable Propagation

<!-- source: AReaL -->

Critical environment variables for distributed training:

```bash
# PyTorch distributed
MASTER_ADDR=<head_node_ip>
MASTER_PORT=<free_port>
WORLD_SIZE=<total_gpus>
RANK=<global_rank>
LOCAL_RANK=<local_gpu_index>

# CPU thread control (set based on allocated cores)
OMP_NUM_THREADS=<cores_per_worker>
MKL_NUM_THREADS=<cores_per_worker>

# NCCL tuning
NCCL_DEBUG=INFO          # Enable for debugging
NCCL_SOCKET_IFNAME=eth0  # Network interface for NCCL
```

### Common Multi-Node Issues

<!-- source: AReaL -->

| Symptom | Likely Cause | First Checks |
|---------|-------------|--------------|
| Job fails to start | Missing environment variables | Check env var propagation, verify launcher logs |
| GPU allocation error | `CUDA_VISIBLE_DEVICES` conflict | Validate device count, check round-robin logic |
| Port binding failure | Port already in use | Use dynamic port allocation, check firewall |
| Worker timeout | Insufficient resources or startup error | Increase timeout (>=60s for large models), check logs |
| Multi-node communication fails | Network misconfiguration | Test connectivity between nodes, check shared storage |

### Best Practices

<!-- source: AReaL -->

- Use dynamic port finding (e.g., `find_free_ports()`) instead of static port assignments
- Ensure shared storage (NFS, S3, etc.) is accessible on all nodes for logs and checkpoints
- Use name resolution services for multi-node service discovery, not hardcoded IPs
- Set `startup_timeout` >= 60 seconds for large model initialization
- Use process tree cleanup on termination to avoid zombie processes

---

## 4. Disaggregated Serving Patterns

### Prefill / Decode Separation

<!-- source: vLLM -->

Some serving systems split prefill and decode into separate instances to improve
throughput and scaling flexibility.

Generic request flow:

1. router selects a compatible prefill instance and decode instance
1. prefill computes KV cache
1. KV cache is transferred to the decode side
1. decode resumes generation without redoing prefill

Operational rules:

- keep request-to-decode routing sticky once a request starts
- separate control-plane address resolution from KV-data transfer
- size KV transfer buffers conservatively so they do not starve normal inference
- if KV overflow is possible, plan a spill path instead of silently dropping cache state

### P2P NCCL Considerations

<!-- source: vLLM -->

Point-to-point NCCL connectors are useful for dynamic P/D pairings, but they add real
memory and topology costs.

Watch for:

- extra NCCL group memory overhead
- symmetric TP assumptions across producer and consumer
- heartbeats and service discovery drift
- `NCCL_MAX_NCHANNELS` materially changing memory footprint

Prefer an async send path over a blocking send path when the implementation offers both.

---

## 5. Session and Log Management

### Service Logs

<!-- source: Ollama -->

When a long-running service behaves unexpectedly, logs usually follow the process
supervisor:

- system service -> use the system journal or service manager logs
- container -> use `docker logs <container>`
- foreground process -> capture stdout/stderr directly
- desktop or packaged runtime -> inspect the product-specific log directory

### Runtime Backend Overrides

<!-- source: Ollama -->

If GPU autodetection is unstable, some runtimes allow forcing a backend or runtime
library through an environment variable. Treat this as a debug workaround, not a normal
configuration path, and document it clearly when used.

---

## 6. Session Management

### tmux (Recommended for Long Training)

```bash
# Create a named session
tmux new-session -s train

# Detach from session: Ctrl+B, then D

# List sessions
tmux list-sessions

# Reattach to session
tmux attach -t train

# Kill a session
tmux kill-session -t train

# Send a command to a running session
tmux send-keys -t train "python train.py" Enter
```

### screen (Alternative)

```bash
# Create a named session
screen -S train

# Detach: Ctrl+A, then D

# Reattach
screen -r train

# List sessions
screen -ls
```

### Why Session Management Matters

- Training runs take hours to days; SSH disconnects will kill foreground processes
- tmux/screen keeps processes alive after disconnection
- Named sessions make it easy to manage multiple concurrent jobs (training, monitoring, debugging)

---

## 5. Common Server-Side Troubleshooting

### Diagnostic Priority Order

When encountering issues, investigate in this order (do not jump to code fixes):

1. **Configuration diff**: Check config files between working and broken runs
2. **Environment differences**: Docker image version, Python deps, env vars
3. **Hardware/network topology**: GPU state, NCCL config, node connectivity
4. **Code changes**: Only after ruling out the above

<!-- source: verl-grounding -->

### Quick Diagnostics

```bash
# System overview
nvidia-smi                              # GPU status
free -h                                  # Memory
df -h                                    # Disk space
nproc                                    # CPU cores

# Network connectivity (multi-node)
ping <other_node>                        # Basic connectivity
nc -zv <node> <port>                     # Port reachability
ip addr show                             # Network interfaces

# Process investigation
ps aux | grep python                     # Find training processes
kill -9 <pid>                            # Force kill a stuck process
fuser -v /dev/nvidia*                    # Who is using GPUs

# Docker diagnostics
docker logs <container>                  # Container logs
docker inspect <container>               # Container config
docker stats                             # Resource usage
```

### Common Failures

| Problem | Quick Fix |
|---------|-----------|
| `CUDA out of memory` | Reduce batch size, enable gradient checkpointing, check for GPU memory leaks |
| `NCCL timeout` | Check network, increase `NCCL_SOCKET_TIMEOUT`, verify `MASTER_ADDR` |
| Training hangs | Check for deadlocks with `py-spy`, verify all ranks are synchronized |
| `RuntimeError: Address already in use` | Kill stale processes, use dynamic port allocation |
| SSH disconnected, process died | Always use tmux/screen for long-running jobs |
| `FileNotFoundError` on shared path | Verify NFS mount, check path exists on all nodes |

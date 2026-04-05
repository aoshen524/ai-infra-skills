<!-- source: FlashAttention -->

# Debugging GPU Kernel Hangs in 2CTA/Cluster Kernels

## General Approach

When a GPU kernel hangs (appears to deadlock), follow this sequence:

1. **Minimal repro**: Strip away everything unrelated. Single batch, single head, fixed seed. The goal is the smallest kernel launch that still hangs.
2. **Printf binary search**: Insert `printf` at coarse granularity (before/after major phases), then narrow down to the exact loop iteration or barrier that never completes.
3. **Identify the deadlock chain**: GPU deadlocks are always cycles. Find which CTA/warpgroup is waiting on what, and trace it back. Common pattern: MMA warpgroup waiting for load warpgroup to fill smem, load warpgroup waiting for MMA warpgroup to consume and release smem slots.
4. **Vary problem size**: Does it hang for all seqlens or only specific ones? Boundary-dependent hangs point to tile scheduling or masking bugs.
5. **Check barrier byte counts**: Incorrect `tx_count` on mbarriers is the single most common cause.
6. **Check phase/parity**: Off-by-one in pipeline phase tracking causes wait-on-wrong-phase deadlocks.

## Printf in CUDA Kernels

### Guards

Unguarded printf from all threads will overflow the buffer and corrupt output. Always guard:

```cuda
// One thread per warp
if (threadIdx.x % 32 == 0) {
    printf("CTA %d warp %d: stage %d iter %d\n",
           blockIdx.x, threadIdx.x / 32, stage, iter);
}

// Single thread per CTA (elect_one)
if (cute::elect_one()) {
    printf("CTA %d: reached barrier %d\n", blockIdx.x, bar_idx);
}
```

### Coarse-to-Fine Strategy

1. First pass: print at phase boundaries (before load, before MMA, before epilogue)
2. Second pass: print inside the loop that doesn't terminate (each iteration)
3. Third pass: print around the specific barrier/wait that blocks

### What to Print

Always include enough context to identify the execution point:
- CTA index (`blockIdx.x`, `blockIdx.y`)
- Warpgroup identity (producer vs consumer, CTA 0 vs CTA 1 in cluster)
- Pipeline stage index
- Loop iteration counter
- Barrier index and expected tx_count

## Deadlock Chain Identification

GPU kernel deadlocks are always circular dependencies. In 2CTA attention kernels, the typical actors are:

- **Producer warpgroup** (load): issues TMA loads, signals mbarrier when data is ready
- **Consumer warpgroup** (MMA): waits on mbarrier, computes, signals completion

A deadlock means:
```
Consumer waiting for mbarrier[stage X] -> Producer hasn't signaled stage X
-> Producer waiting for consumer to release stage (X - num_stages) % num_stages
-> Consumer never released because it's stuck waiting for stage X
```

To break the chain: verify that every `arrive` has a matching `wait`, and that `tx_count` matches the actual bytes transferred.

## Barrier Byte Counts (tx_count)

TMA-based pipelines use `mbarrier.arrive_and_expect_tx` to track how many bytes will be deposited into shared memory before the barrier is satisfied.

### Critical Rule for 2CTA Mode

In 2CTA (cluster) kernels, both CTAs in the cluster signal the same barrier. The tx_count must account for contributions from ALL CTAs:

```
tx_count = bytes_per_tile * num_tiles * cta_group_size
```

If `cta_group_size = 2` but you set `tx_count` for only 1 CTA, the barrier will never be satisfied -- the consumer waits forever.

### Common Mistakes

- Forgetting to multiply by `cta_group_size` when both CTAs load into the same smem region
- Setting tx_count for the wrong buffer (e.g., K buffer size when loading V)
- Not accounting for swizzle padding in the byte count

## Phase/Parity Tracking

`mbarrier_try_wait_parity` uses a single parity bit (0 or 1), not the full phase counter. When the pipeline wraps around:

```cuda
// Correct: use phase % 2
int parity = phase % 2;
mbarrier_try_wait_parity(&barrier[stage], parity);

// WRONG: using phase directly when phase > 1
mbarrier_try_wait_parity(&barrier[stage], phase);  // UB for phase >= 2
```

Off-by-one in phase initialization (starting at 0 vs 1) can cause the first or last iteration to wait on the wrong parity.

## Compiler as Bug Source

### Symptom

Kernel works correctly WITH printf statements but hangs WITHOUT them. This is a strong indicator that printf is acting as an implicit memory/execution barrier, and the compiler is reordering or optimizing away a necessary fence.

### Signs

- Adding `printf("")` (empty string, never actually printed) to a specific location prevents the hang
- Different optimization levels (`-O0` vs `-O2`) change behavior
- Adding `__threadfence()` or `asm volatile("" ::: "memory")` at the printf location also fixes it

### Workarounds

1. Insert explicit fence: `__threadfence()` or `asm volatile("" ::: "memory")`
2. In DSL frameworks (like CuTe/Triton), use `@dsl_user_op` to inject inline PTX with compiler barriers
3. Compare PTX output with and without the printf to identify what the compiler moved

## 2CTA-Specific Pitfalls

### tcgen05.commit with Empty Commit Groups

On SM100+, `tcgen05.commit` must not be issued if the commit group is empty (no prior `tcgen05.mma` in that group). An empty commit is undefined behavior and can cause hangs or silent corruption.

### Producer Tail Deadlock

When the producer finishes its tiles before the consumer, it must still participate in barrier signaling for remaining stages. If the producer exits its loop early without draining the pipeline, the consumer blocks forever.

```cuda
// Producer must signal remaining stages even if no data to load
for (int i = current_stage; i < num_stages; i++) {
    arrive_no_data(&barrier[i]);  // signal with 0 bytes
}
```

### Tile Scheduler and Cluster Shape

The tile scheduler (work distribution across CTAs) must ensure that the number of tiles along each dimension is divisible by `cluster_shape`. If `num_tiles_m % cluster_m != 0`, some CTAs in the cluster get no work but still need to participate in cluster-level barriers.

### Cross-CTA vs Per-CTA Pipelines

In 2CTA kernels, carefully distinguish:
- **Shared pipeline**: both CTAs in the cluster use the same smem barriers (e.g., for K/V that both CTAs read)
- **Per-CTA pipeline**: each CTA has its own barriers (e.g., for Q tiles that are CTA-local)

Mixing these up causes either double-counting of tx_bytes or missed signals.

### Softmax Masking Offset

In causal attention with 2CTA tiling, the causal mask offset must account for which CTA within the cluster owns which row range. Using the global tile index instead of the per-CTA tile index shifts the mask, producing wrong results (not a hang, but a correctness bug that often surfaces during hang debugging).

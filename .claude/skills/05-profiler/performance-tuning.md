# Performance Tuning: Block Sizes, SASS Analysis, and Kernel Optimization

<!-- source: flash-attention, flashinfer -->

Low-level performance tuning methodology for GPU kernels, focusing on SM90+ (Hopper) architectures. Covers tile/block size selection, SASS disassembly for MMA instruction analysis, and resource constraint reasoning.

## Block Size Tuning (SM90 / Hopper)

<!-- source: flash-attention -->

### Hardware Constraints (H100)

| Resource | Limit | Notes |
|----------|-------|-------|
| **SMEM** | 228 KB total | ~224 KB usable after reserving ~3 KB for LSE, dPsum, mbarriers |
| **Registers** | 2 WG: 216 usable/thread, 3 WG: 128 usable/thread | After overhead (24 regs for 2 WG, 32 for 3 WG) |
| **GMMA atom M** | Always 64 | Effective M dimension must be divisible by 64 |
| **GMMA atom N** | `atom_layout_n * 8` | N dimension must be divisible by this |

### Warp Group Architecture (SM90)

Each kernel has `num_wg + 1` warp groups (128 threads each):
- **WG0**: Producer -- TMA loads
- **WG1** (backward only): Producer -- dQaccum store
- **Remaining WGs**: MMA consumers -- all GEMMs

For forward: `tile_m = num_wg * 64`.

### Key Design Decisions

#### 1. Number of Warp Groups (num_wg)

| num_wg | tile_m (fwd) | Threads | Reg budget/thread | Best for |
|--------|-------------|---------|-------------------|----------|
| 2 | 128 | 384 | 216 | hdim <= 128 |
| 3 | 192 | 512 | 128 | hdim 129-192 |

More WGs = larger tile_m = better M-direction parallelism, but tighter register budget and higher SMEM usage.

#### 2. swap_AB

Each MMA can swap its A and B operands, transposing the output tile. This exchanges which dimension maps to M (must be divisible by 64) and which maps to N.

**When to swap:**
- If the natural M dimension is not divisible by 64 but N is
- To change which operand is in registers vs shared memory

**Forward**: No swap needed since `tile_m = num_wg * 64` is always divisible by 64.

**Backward** (5 MMAs):
- **SdP** (S=Q@K^T): Swap if `tile_m % 64 != 0`
- **dKV** (dK=dS^T@Q): Swap if `tile_n % 64 != 0` but `hdim % 64 == 0`
- **dQ** (dQ=dS@K): Swap if `tile_m % 64 != 0` but `hdim % 64 == 0`

#### 3. AtomLayout

Distributes WGs across M and N dimensions of MMA output:
- More WGs in N direction (`wg_n` larger) = each instruction reads a smaller B slice, but more total SMEM traffic from overlapping A reads
- Typically **smaller wg_n = less total SMEM traffic**

#### 4. Register-Source Optimization (mma_dkv_is_rs)

When conditions are met (`AtomLayoutMSdP == 1 && AtomLayoutNdKV == num_wg && SdP_swapAB && !dKV_swapAB`), P and dS matrices can be kept in registers and fed directly as A operand for dV and dK GEMMs. This:
- Eliminates sP from SMEM (saves `tile_m * tile_n * 2` bytes)
- Eliminates P register-to-SMEM store traffic
- Eliminates A operand reads for dK/dV GEMMs

Always preferred when conditions are met.

#### 5. Pipeline Staging

**Forward**: Q (1 stage), K/V (2 stages, double-buffered with TMA), O overlaps with Q in SMEM.

**Backward**: Q (2 stages), dO (2 stages if SMEM allows, else 1), PdS (1 stage), K/V (persistent in SMEM).

### Resource Accounting

#### Register Budget

Accumulator registers per thread per WG = `M * N / (num_wg * 128)`.

**Forward peak**: `regs_S + regs_P + regs_O` (with WG overlap) or `regs_S + regs_O` (without). `regs_P = regs_S / 2` (bf16 vs f32).

**Backward peak**: `max(2 * regs_SdP, regs_dQ) + regs_dK + regs_dV`. S and dP accumulators are both live; dQ reuses S+dP space after consumption.

#### SMEM Budget

**Forward**: `max(sQ, sO) + sK*2 + sV*2 + sP`
- sQ = `tile_m * hdim * 2`, sK = `tile_n * hdim * 2 * 2` (stages), sP = `tile_m * tile_n * 2` (0 if RS)

**Backward**: `sQ*2 + sK + sV + sdO*dO_stage + sP + sdS + sdQaccum`
- sdQaccum = `tile_m * hdim * 4` (f32)

#### SMEM Traffic

Normalize as traffic per block (`traffic / (tile_m * tile_n)`) for comparison across tile sizes. Lower is better.

### Config Search Tool

```bash
# Enumerate feasible configs for a given head dimension
python config_search.py --headdim 128

# Forward only
python config_search.py --mode fwd --headdim 192-128

# Backward with custom tile choices
python config_search.py --mode bwd --headdim 192 --tile-m 64,80 --tile-n 64,96
```

### Example Configurations

| Config | tile_m | tile_n | WG | SMEM | Notes |
|--------|--------|--------|----|------|-------|
| hdim=128 fwd | 128 | 192 | 2 | 224K | Best config, RS enabled |
| hdim=128 bwd | 80 | 128 | 2 | 204K | mma_dkv_is_rs=True |
| hdim=192 bwd | 64 | 96 | 3 | 216K | Only feasible tile_n > 64 for hdim=192 |
| hdim=192/128 (DeepSeek) | 64 | 96 | 3 | 212K | AtomLayoutNdKV=3 needed |

## SASS Analysis for HGMMA Instructions

<!-- source: flash-attention -->

SASS (Shader Assembly) analysis lets you verify that the compiled kernel matches expected MMA patterns.

### Dumping SASS

```bash
# Compile with cubin output
CUTE_DSL_KEEP_CUBIN=1 python -c "..."

# Find the cubin
ls *.cubin

# Disassemble and extract HGMMA instructions
nvdisasm kernel.sm_90a.cubin | grep "HGMMA\."
```

### Reading HGMMA Instructions

Each `HGMMA.MxNxK.F32.BF16` instruction is one warpgroup-level MMA:
- **M**: always 64 (one warpgroup = 128 threads, fixed by hardware)
- **N**: output columns per instruction (e.g., 96, 128, 192)
- **K**: always 16 for BF16 (one K-step per instruction)

### RS vs SS: Source Operand Mode

The 2nd operand (operand A) reveals the data source:

| Pattern | Mode | Meaning |
|---------|------|---------|
| `gdesc[UR..]` | SS | Both A and B from shared memory |
| `R<N>` (plain register) | RS | A from registers, B from shared memory |
| `gdesc[UR..].tnspA.tnspB` | SS transposed | Transposed views of SMEM |

**RS reduces SMEM traffic**: if data is already in registers from a previous computation, RS feeds it directly to the next GEMM without writing to and reading from shared memory.

### Identifying GEMMs from SASS

1. **Same dest register** across consecutive HGMMA instructions = accumulating into the same output tile (iterating over K dimension)
2. **Count of instructions with same dest register** x 16 = **reduction (K) dimension**
3. **Different dest registers** in interleaved pattern = a large MMA split into multiple 64-row parts (M = num_parts x 64)

### Verification Table Template

| GEMM | Atom | # Acc | I/acc | M | N | K | RS/SS | Check |
|------|------|-------|-------|---|---|---|-------|-------|
| S = Q @ K.T | 64xNx16 | 1 | hdim/16 | tile_m | tile_n | hdim | SS | |
| dQ = dS @ K | 64xNx16 | 1 | tile_n/16 | tile_m | hdim | tile_n | RS? | |

## General Optimization Strategies

### Occupancy vs. Resource Usage

- More SMEM per block = fewer concurrent blocks = lower occupancy
- More registers per thread = fewer concurrent warps
- Balance: target the minimum occupancy that hides memory latency

### Memory Access Patterns

- Ensure coalesced global memory access (consecutive threads access consecutive addresses)
- Use TMA (Tensor Memory Accelerator) on SM90+ for efficient bulk data movement
- Double-buffer shared memory loads to overlap computation and memory

### Numerical Precision

- Use FP32 accumulators for FP16/BF16 MMA operations
- Be explicit about accumulation precision in reduction operations
- Check for potential overflow in large reductions

### CUDA Graph Compatibility

- Keep CUDA graph enabled for performance benchmarking by default
- Disable only when debugging compatibility issues
- Some profiling features (tensor statistics, dumps) are automatically skipped during CUDA graph capture

## Serving-Path Integration Check

For inference frameworks, do not stop at kernel-level tuning.

Before claiming a serving optimization worked, verify:

- the optimized path still sits inside the `torch.compile` region when relevant
- custom op registration did not introduce extra graph breaks
- profiler evidence still shows the intended path in the real server trace

Use `torch._dynamo.explain` or the framework's graph-break tooling when operator-level
gains disappear after integration.

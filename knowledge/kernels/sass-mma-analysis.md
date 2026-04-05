<!-- source: FlashAttention -->

# Analyzing SASS for HGMMA (Hopper MMA) Instructions

## Dumping SASS

To inspect the actual machine instructions emitted for a kernel:

```bash
# Compile to cubin (if not already)
nvcc -arch=sm_90a -cubin kernel.cu -o kernel.cubin

# Disassemble and filter for MMA instructions
nvdisasm kernel.cubin | grep "HGMMA\."

# For a specific kernel in a .so/.fatbin
cuobjdump -sass library.so | grep "HGMMA\."
```

If you have a `.cubin` embedded in a Python package, extract it first:
```bash
cuobjdump -xelf all library.so   # extracts .cubin files
nvdisasm extracted.cubin | grep "HGMMA\."
```

## Reading HGMMA Instructions

An HGMMA instruction looks like:
```
HGMMA.64x128x16.F32.BF16.BF16  R24, gdesc[UR4][UR5], R48, ...
```

### Decoding the Mnemonic

- `HGMMA` = Hopper Generalized Matrix Multiply-Accumulate
- `64x128x16` = MxNxK tile dimensions
  - M = 64 always (warpgroup-level, 4 warps x 16 threads contributing rows)
  - N = output columns per instruction (64, 128, 192, 256)
  - K = reduction dimension per instruction (16 for BF16/FP16, 8 for TF32)
- `F32` = accumulator datatype
- `BF16.BF16` = A and B operand datatypes

### RS vs SS Mode

The operand encoding reveals the data source:

- **SS mode** (shared-shared): `gdesc[UR..]` = smem descriptor for both A and B. Data read from shared memory via TMA descriptors.
  ```
  HGMMA.64x128x16.F32.BF16.BF16  R24, gdesc[UR4][UR5], gdesc[UR6][UR7], ...
  ```
- **RS mode** (register-shared): `R<N>` for A operand = registers, `gdesc[UR..]` for B = smem. A operand is pre-loaded into registers, reducing smem traffic.
  ```
  HGMMA.64x128x16.F32.BF16.BF16  R24, R48, gdesc[UR6][UR7], ...
  ```

RS mode is preferred when the A operand is reused across multiple HGMMA instructions (e.g., Q matrix in attention), as it avoids redundant smem reads.

## Identifying GEMMs from SASS

### Same Destination Register = Accumulating K

Multiple HGMMA instructions writing to the same destination register set are accumulating along the K dimension:

```
HGMMA.64x128x16.F32.BF16  R24, ...   # K iteration 0
HGMMA.64x128x16.F32.BF16  R24, ...   # K iteration 1 (accumulates into R24)
HGMMA.64x128x16.F32.BF16  R24, ...   # K iteration 2
```

Count of instructions x K_per_instruction = total K dimension:
- 3 instructions x 16 = K=48

### Different Destination Registers = Split M

When HGMMA instructions are interleaved with different destination register groups, this indicates the M dimension is split into multiple parts:

```
HGMMA.64x128x16.F32.BF16  R24, ...   # M part 0, K iter 0
HGMMA.64x128x16.F32.BF16  R56, ...   # M part 1, K iter 0
HGMMA.64x128x16.F32.BF16  R24, ...   # M part 0, K iter 1
HGMMA.64x128x16.F32.BF16  R56, ...   # M part 1, K iter 1
```

Total M = num_parts x 64 (since each HGMMA always does M=64).

## Case Study: BWD SM90 hdim=192 hdimv=128

Backward pass attention kernel with head_dim=192 (Q/K) and head_dim_v=128 (V/O).

### SASS Analysis

Total HGMMA instructions: 47

| GEMM | Operation | M | N | K | Instruction Count | Verification |
|------|-----------|---|---|---|-------------------|--------------|
| dS = dO * V^T | dS computation | 128 | 192 | 128 | 2 parts x 8 K-iters = 16 | M=2x64=128, K=8x16=128 |
| dV = dO^T * S | dV accumulation | 128 | 128 | 128 | 2 parts x 8 K-iters = 16 | M=2x64=128, K=8x16=128 |
| dQ = dS * K | dQ computation | 128 | 192 | variable | variable | Depends on seqlen |
| dK = dS^T * Q | dK accumulation | 128 | 192 | variable | variable | Depends on seqlen |

### Verification Method

1. Count total HGMMA instructions: `nvdisasm kernel.cubin | grep -c "HGMMA\."`
2. Group by destination register to identify distinct GEMMs
3. Count K iterations per GEMM (same dest reg group)
4. Verify: K_iterations x 16 = expected K dimension
5. Count dest reg groups: num_groups x 64 = expected M dimension
6. Cross-check N from the instruction mnemonic

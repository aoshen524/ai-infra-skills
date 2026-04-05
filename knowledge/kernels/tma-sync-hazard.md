<!-- source: FlashAttention -->

# compute-sanitizer Racecheck False Positives with cp.async.bulk

## Summary

`compute-sanitizer --tool racecheck` reports false-positive data races when kernels use raw-address TMA (`cp.async.bulk`) but NOT when using descriptor-based TMA (`cp.async.bulk.tensor`). The races are false positives -- the kernel produces correct results and the mbarrier synchronization is valid. The sanitizer simply cannot model the synchronization correctly for the raw-address variant.

## Root Cause

The sanitizer internally attributes the shared memory write to different entities depending on the TMA variant:

- **Raw-address TMA** (`cp.async.bulk`): The sanitizer attributes the smem write to the issuing thread/warp. When the same warp later reads the data (after waiting on an mbarrier), the sanitizer sees the same thread writing then reading and considers it a potential race if there's a loop back-edge between them.
- **Descriptor-based TMA** (`cp.async.bulk.tensor`): The sanitizer treats the smem write as an external DMA operation, not attributed to any specific thread. The mbarrier wait is correctly recognized as a synchronization point.

The mbarrier synchronization is correct in both cases. The sanitizer simply doesn't model the happens-before relationship across dynamic loop back-edges for the raw-address variant.

## Instruction Comparison Table

| Instruction | Scope | Sanitizer Result |
|---|---|---|
| `cp.async.bulk.shared::cluster.global.mbarrier::complete_tx::bytes` | CTA scope, raw address | **HAZARD reported** |
| `cp.async.bulk.shared::cluster.global.mbarrier::complete_tx::bytes.multicast::cluster` | Cluster scope, raw address | **HAZARD reported** |
| `cp.async.bulk.tensor.1d.shared::cluster.global.mbarrier::complete_tx::bytes.tile` | Descriptor, 1D tile | Clean |
| `cp.async.bulk.tensor.2d.shared::cluster.global.mbarrier::complete_tx::bytes.tile` | Descriptor, 2D tile | Clean |
| `cp.async.bulk.tensor.3d.shared::cluster.global.mbarrier::complete_tx::bytes.tile` | Descriptor, 3D tile | Clean |

## Proof That It Is a False Positive

Multiple lines of evidence confirm the races are not real:

1. **Data correctness**: Output matches reference implementation bit-for-bit across all test cases
2. **Single-warp test**: Running with a single warp (no opportunity for actual races) produces clean racecheck output, yet the multi-warp version reports hazards on the same data
3. **Unrolled loop**: Manually unrolling the pipeline loop eliminates the reported hazard (sanitizer can see the happens-before across straight-line code)
4. **Explicit bar.sync**: Inserting a redundant `bar.sync` (which should be unnecessary given the mbarrier) makes the hazard disappear, confirming the sanitizer needs explicit sync it can track
5. **Descriptor TMA is clean**: Replacing only the TMA instruction (same barriers, same data flow) eliminates all reports

## Fix

Switch from raw-address TMA to descriptor-based TMA for the affected shared memory buffers:

- Replace `CopyBulkG2SOp` (raw address) with `CopyBulkTensorTileG2SOp` (descriptor-based)
- This requires constructing a TMA descriptor for the source tensor, but the CuTe/CUTLASS helpers make this straightforward
- No changes to barrier logic, pipeline structure, or smem layout needed

This is the correct fix even beyond silencing the sanitizer -- descriptor-based TMA is generally preferred on Hopper+ for better hardware scheduling.

## Important Note on Suppression Flags

The flag `--racecheck-memcpy-async=no` does NOT help with this issue. That flag targets the older `cp.async` instruction (Maxwell-Ampere era), not the newer `cp.async.bulk` (Hopper+). There is no equivalent suppression flag for `cp.async.bulk` as of CUDA 12.x.

## Diagnostic Checklist

When you see racecheck reports in a TMA-based kernel:

1. Check which TMA variant is used (raw vs descriptor) -- look at the PTX
2. If raw-address: check if the reported addresses are in smem buffers filled by TMA
3. Verify the mbarrier wait/arrive pattern is correct
4. Run correctness tests -- if results are bit-exact, likely false positive
5. Try single-warp execution to confirm
6. Switch to descriptor TMA as the proper fix

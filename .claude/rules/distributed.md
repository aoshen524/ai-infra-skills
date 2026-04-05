# Rule: Distributed Training Code Patterns
<!-- sources: AReaL, torchtitan -->

## Process Group Management
<!-- source: AReaL -->

- **Never create global process groups** in module-level code
- Always pass `process_group` explicitly; don't rely on the default group
- Use `dist.get_rank(group)` not `dist.get_rank()` when the group matters
- Clean up process groups in `__del__` or a context manager

## DeviceMesh & DTensor
<!-- source: AReaL, torchtitan -->

- Never assume a 1D mesh. Always assert on mesh dimensions before using them
- DTensor requires a consistent mesh across all ranks
- Use `DTensor.from_local()` with correct placements
- Validate tensor placements (`Replicate`, `Shard`, `Partial`) explicitly
- For `DTensor.to_local`, always specify `grad_placements` explicitly, especially when the original DTensor has `Replicate` or `Partial` placement. This includes indirect calls from `local_map` (`in_grad_placements`) and `full_tensor` (`grad_placements`)
- When enforcing a field is not None for plain tensor inputs, do so with a clear error message

## Communication Patterns
<!-- source: AReaL -->

- **All-reduce**: Must be called by all ranks in the group
- **Broadcast**: Specify `src` rank explicitly
- **Barrier**: Avoid unless necessary (debugging only)
- Check `NCCL_ASYNC_ERROR_HANDLING` for deadlock debugging

## Consider All Parallelism Combinations
<!-- source: torchtitan -->

- When adding or modifying distributed code, think through how it interacts with every parallelism dimension (DP, TP, PP, CP, EP, etc.)
- A fix for one parallelism configuration may break another
- Include the parallelism configuration in bug reports and test descriptions

## Model-Agnostic vs Model-Specific Code
<!-- source: torchtitan -->

- Helper functions that apply to any model (e.g., async TP, tensor parallel utilities, generic sharding) belong in a shared distributed utilities module, not in model-specific files
- Model-specific parallelization strategies belong alongside the model definition

## Be Conservative with Changes
<!-- source: torchtitan -->

- Distributed training code is hard to test exhaustively
- When modifying existing behavior:
  - Verify numerics match before and after across multiple parallelism configs
  - Watch for silent correctness issues (wrong gradient placements, identity operations that break checkpointing)
  - If changing something that's been converged and validated, provide strong justification and thorough testing

## Common Pitfalls
<!-- source: AReaL -->

| Issue         | Cause                            | Fix                              |
| ------------- | -------------------------------- | -------------------------------- |
| Hang          | Mismatched collective calls      | Ensure all ranks call same op    |
| Wrong results | Incorrect reduction op           | Check `ReduceOp` (SUM vs MEAN)  |
| OOM           | Unsharded tensor on wrong device | Verify DTensor placements        |

## Debugging
<!-- source: AReaL -->

- Set `TORCH_DISTRIBUTED_DEBUG=DETAIL` for verbose logging
- Use `NCCL_DEBUG=INFO` for NCCL-level issues

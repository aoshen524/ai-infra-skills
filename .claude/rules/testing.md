# Rule: Testing Strategy for AI Infrastructure
<!-- sources: AReaL, torchtitan, SGLang, SLIME -->

## Pytest Markers
<!-- source: AReaL -->

| Marker                                  | When to Use          |
| --------------------------------------- | -------------------- |
| `@pytest.mark.slow`                     | Takes > 10 seconds   |
| `@pytest.mark.asyncio`                  | Async test functions  |
| `@pytest.mark.skipif(cond, reason=...)` | Conditional skip      |
| `@pytest.mark.parametrize(...)`         | Parameterized tests   |

## Test Structure
<!-- source: AReaL -->

```python
def test_<what>_<condition>_<expected>():
    """Test that <what> does <expected> when <condition>."""
    # Arrange
    ...
    # Act
    ...
    # Assert
    ...
```

## GPU Test Constraints
<!-- source: AReaL -->

- **Always skip gracefully** when GPU is unavailable:
  ```python
  CUDA_AVAILABLE = torch.cuda.is_available()

  @pytest.mark.skipif(not CUDA_AVAILABLE, reason="CUDA not available")
  def test_gpu_feature():
      ...
  ```
- Clean up GPU memory: `torch.cuda.empty_cache()` in fixtures
- Use the smallest possible model/batch size for unit tests
- Check GPU availability before running GPU-heavy tests; don't assume GPUs are free

## Mocking Distributed
<!-- source: AReaL -->

- Use `torch.distributed.fake_pg` for unit tests
- Mock `dist.get_rank()` and `dist.get_world_size()` explicitly
- Don't mock internals of FSDP/DTensor -- use integration tests for those

## Distributed Test Requirements
<!-- source: torchtitan -->

- Include the parallelism configuration in test descriptions
- When testing distributed changes, verify numerics across multiple parallelism configs
- Watch for silent correctness issues (wrong gradient placements, identity operations)

## Fixtures
<!-- source: AReaL -->

- Prefer `tmp_path` over manual temp directories
- Use `monkeypatch` for environment variables
- Scope expensive fixtures appropriately (`session` > `module` > `function`)

## Test Harness Discipline
<!-- source: SGLang -->

- If the repo provides a custom base test case, use it instead of raw `unittest.TestCase`
- Class-level teardown must be defensive when setup can fail partway through
- Prefer mock-based unit tests for routing, config, middleware, and argument parsing logic
- Launch real servers or engines only when the behavior genuinely depends on runtime inference

## Assertions
<!-- source: AReaL -->

- Use `torch.testing.assert_close()` for tensor comparison
- Specify `rtol`/`atol` explicitly for numerical tests
- Avoid bare `assert tensor.equal()` -- it gives no useful error message on failure

## Experiments Testing
<!-- source: torchtitan -->

- Experimental code must still pass linting (`pre-commit run --all-files`)
- Experiment tests should not depend on or modify core module behavior

## CI Wiring
<!-- source: SLIME, SGLang -->

- Register new tests in the lightest CI suite that still exercises the target behavior
- If CI uses generated workflow files, edit the template and regenerate the outputs
- Document GPU assumptions, exact commands, and any model/server dependency in the PR

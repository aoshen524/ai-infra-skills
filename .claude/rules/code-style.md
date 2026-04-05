# Rule: Code Style for AI Infrastructure
<!-- sources: AReaL, torchtitan -->

## Design Patterns
<!-- source: AReaL -->

- **Prefer composition over inheritance**: Avoid deep class hierarchies
  - Good: `Engine` holds a `Checkpointer` instance
  - Avoid: `CheckpointableEngine(Engine)` -> `FSDPCheckpointableEngine(CheckpointableEngine)`
- Keep inheritance shallow (2 levels max when possible)
- Use mixins sparingly; prefer explicit delegation

## Naming Conventions
<!-- source: AReaL, torchtitan -->

| Type             | Pattern       | Example                             |
| ---------------- | ------------- | ----------------------------------- |
| Config dataclass | `XxxConfig`   | `GRPOConfig`, `FSDPConfig`          |
| Engine class     | `XxxEngine`   | `FSDPEngine`, `InferenceEngine`     |
| Workflow class   | `XxxWorkflow` | `RLWorkflow`, `MultiTurnWorkflow`   |
| Reward function  | `xxx_reward`  | `math_reward`, `code_reward`        |
| Count fields     | `num_` prefix | `num_workers`, `num_layers`         |

## Performance Patterns
<!-- source: AReaL -->

- **Avoid GPU-CPU sync**: `.item()`, `.tolist()`, `print(tensor)` cause sync -- never use in hot paths
- **Prefer batch operations**: Avoid Python loops over tensor elements
- **In-place ops**: Use when safe, but be careful with autograd (`.add_()` vs `+`)

## Tensor Conventions
<!-- source: AReaL -->

- Shape convention: `[batch, seq_len, hidden]` or document clearly if different
- Use `torch.Size` assertions for shape validation in debug builds
- Prefer explicit dtype/device over implicit conversion

## Logging
<!-- source: AReaL -->

- Use a project-level logger utility, NOT `print` or bare stdlib `logging`
- Use PascalCase descriptive names for loggers (e.g., `getLogger("RLWorkflow")`)
- For per-rank loggers: `[{Component} Rank {N}]` format
- Log levels:
  - DEBUG: Detailed tracing (avoid in hot paths)
  - INFO: Milestones (training start, checkpoint saved)
  - WARNING: Recoverable issues
  - ERROR: Failures requiring attention

## Import Style
<!-- source: AReaL -->

- Group: stdlib, third-party, project-internal (let formatter handle order)
- Never use `from x import *`
- Prefer explicit imports over module-level imports for large modules

## Experiments Code
<!-- source: torchtitan -->

- Experimental code has more flexibility than core, but must still pass linting
- Never modify core modules to accommodate experiment needs -- work around it in the experiment folder
- Keep distinct experimental features in separate folders; don't bundle unrelated functionality

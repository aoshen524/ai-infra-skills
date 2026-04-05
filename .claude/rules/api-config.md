# Rule: API and Config Conventions
<!-- sources: AReaL, torchtitan, Megatron-LM -->

## Dataclass Conventions
<!-- source: AReaL -->

```python
@dataclass
class XxxConfig:
    """One-line description.

    Attributes:
        field_name: Description with default explained.
    """
    # Required fields first (no default)
    required_field: str

    # Optional fields with defaults
    optional_field: int = 32

    # Internal fields last (underscore prefix)
    _internal: str = field(default="", repr=False)
```

## Field Ordering
<!-- source: AReaL, torchtitan -->

1. Required fields (no default) -- never use `None` as a default for truly required fields
2. Common, frequently-used optional fields
3. Advanced/rare optional fields
4. Internal fields (`_prefix`)
5. Prefer keyword-only arguments after the first positional arg to prevent positional mistakes

## Naming Conventions
<!-- source: torchtitan -->

- Use descriptive names that reflect what the config controls
- Use `num_` prefix for count fields (e.g., `num_workers`, `num_layers`)
- Prefer strings as config names when they benefit extensibility (e.g., model selection, dataset selection)
- Use `Literal` for enum-like choices

## Validation
<!-- source: AReaL -->

- Use `__post_init__` for validation
- Raise `ValueError` with a clear message:
  ```python
  def __post_init__(self):
      if self.batch_size <= 0:
          raise ValueError(f"batch_size must be positive, got {self.batch_size}")
  ```

## Safety
<!-- source: torchtitan -->

- When config values are passed through to external APIs (e.g., DataLoader kwargs), ensure all values are safe for all code paths (e.g., what happens when `num_workers == 0`?)
- When a config option silently doesn't take effect in certain code paths, emit a warning to the user

## Backward Compatibility
<!-- source: AReaL, Megatron-LM -->

- **Adding fields**: Add with default value (safe)
- **Removing fields**: Deprecate first, remove in next major version
- **Renaming fields**: Add new field, keep old with deprecation warning
- **Changing types**: Avoid; use `Union` if necessary
- **Adding required parameters** to public functions is breaking
- **Reordering public parameters** is breaking even if names stay the same
- If an API is unstable, label it clearly as internal or experimental instead of changing it silently

## Public API Review
<!-- source: Megatron-LM -->

For public APIs and config entrypoints:

- removing a parameter is breaking
- adding a parameter without a default is breaking
- making an optional parameter required is breaking
- removing a public function is breaking

Safe changes usually include:

- adding optional parameters with defaults
- adding new public functions
- making required parameters optional

## CLI Integration
<!-- source: AReaL, torchtitan -->

- Fields exposed to CLI must have clear help text in metadata
- Avoid complex nested types in CLI-exposed configs
- Use the project's existing config and job config infrastructure; don't introduce custom argument parsing or parallel config systems

## Documentation
<!-- source: AReaL -->

- All public configs must have a docstring
- Document constraints (e.g., "must be power of 2")
- Include example values for non-obvious fields

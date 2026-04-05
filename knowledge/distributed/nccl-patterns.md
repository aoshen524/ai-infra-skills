<!-- source: NCCL -->

# NCCL Contribution and Development Patterns

## Code Style

NCCL follows a consistent code style throughout the codebase:

- **Indentation**: 2-space indent (no tabs)
- **Line length**: 100 character maximum
- **Braces**: K&R style (opening brace on same line as statement)
- **Function naming**: `ncclCamelCase` for public functions, `camelCase` for internal
- **Macro naming**: `UPPER_CASE` with `NCCL` prefix for public macros
- **Type naming**: `ncclCamelCase_t` for types (e.g., `ncclComm_t`, `ncclResult_t`)
- **Comments**: C-style `/* */` for multi-line, `//` for single-line

```c
// Function naming example
ncclResult_t ncclAllReduce(const void* sendbuff, void* recvbuff,
                           size_t count, ncclDataType_t datatype,
                           ncclRedOp_t op, ncclComm_t comm,
                           cudaStream_t stream) {
  // implementation
}
```

## Return Codes

All NCCL functions return `ncclResult_t`. Never use raw return values or exceptions.

### Guard Macros

```c
// For NCCL internal calls
NCCLCHECK(ncclSomeInternalFunction(args));
// Expands to: if (ret != ncclSuccess) { ... log and return ret; }

// For CUDA calls
CUDACHECK(cudaMalloc(&ptr, size));
// Expands to: if (ret != cudaSuccess) { ... log and return ncclUnhandledCudaError; }

// For system calls (malloc, socket, etc.)
SYSCHECK(posixCall(args), "description");
// Expands to: if (ret != 0) { ... log errno and return ncclSystemError; }
```

### Error Propagation Pattern

```c
ncclResult_t ncclMyFunction(ncclComm_t comm) {
  ncclResult_t ret = ncclSuccess;

  // Early validation
  if (comm == NULL) return ncclInvalidArgument;

  // Chained calls with NCCLCHECK
  NCCLCHECK(ncclStep1(comm));
  NCCLCHECK(ncclStep2(comm));

  return ret;
}
```

Never silently swallow errors. Every error path should either handle the error or propagate it upward.

## Memory Management

### Allocation

Always use NCCL's own allocator for consistency and tracking:

```c
// Preferred: ncclCalloc (zero-initialized)
struct ncclSomething* obj;
NCCLCHECK(ncclCalloc(&obj, count));

// NOT: malloc/calloc directly
// NOT: new (this is C, not C++)
```

### Memory Fences

When sharing data between CPU threads or between CPU and GPU:

```c
// Store fence before publishing data
__atomic_store_n(&sharedFlag, 1, __ATOMIC_RELEASE);

// Load fence when consuming data
int val = __atomic_load_n(&sharedFlag, __ATOMIC_ACQUIRE);
```

### Deallocation

Free resources in reverse order of allocation to avoid dangling references:

```c
ncclResult_t ncclCleanup(ncclComm_t comm) {
  // Free in reverse order
  free(comm->channels);      // allocated last
  free(comm->topology);      // allocated second
  free(comm->bootstrap);     // allocated first
  free(comm);
  return ncclSuccess;
}
```

## CUDA-Specific Patterns

### Register Usage

NCCL kernels are sensitive to register pressure since many CTAs must run concurrently for latency hiding:

```c
// Document launch config
// Grid: (nChannels, 1, 1)  Block: (NCCL_MAX_NTHREADS, 1, 1)
// Registers: target < 40 per thread for high occupancy
__global__ void ncclKernel(...) {
  // Minimize local variables
  // Use shared memory for temporaries
}
```

### Warp Divergence

Avoid warp divergence in performance-critical paths:

```c
// BAD: divergent branch
if (threadIdx.x < 16) {
  doPath1();
} else {
  doPath2();
}

// BETTER: uniform branch (all threads in warp take same path)
if (warpId < nWarpsForPath1) {
  doPath1();
} else {
  doPath2();
}
```

### Kernel Launch

Prefer `cudaLaunchKernel` over `<<<>>>` syntax for better error handling:

```c
void* args[] = {&sendbuff, &recvbuff, &count};
CUDACHECK(cudaLaunchKernel(
    (void*)ncclKernel,
    dim3(nChannels, 1, 1),
    dim3(nThreads, 1, 1),
    args, smemSize, stream));
```

## Contribution Workflow

### Process

1. **Open an issue first**: Describe the problem or feature. Discuss with maintainers before writing code.
2. **Design discussion**: For non-trivial changes, write a brief design doc (in the issue) covering approach, alternatives considered, and performance implications.
3. **Review**: Maintainers review design before implementation begins.
4. **Implementation**: Follow code style, add tests, update documentation.
5. **PR submission**: Reference the issue, include benchmark results if performance-related.

### What Makes a Good NCCL PR

- Benchmarks showing performance impact (even if neutral -- "no regression")
- Test coverage for new code paths
- Documentation updates for new APIs or behavior changes
- Minimal scope: one logical change per PR

## Commit Messages

### Format

```
Short imperative summary (50 chars or less)

Problem: What's wrong / what's missing
Solution: What this commit does
Limitations: Known limitations or follow-up work

Signed-off-by: Name <email>
```

### Rules

- **Imperative mood**: "Fix race condition" not "Fixed race condition"
- **Title**: 50 characters max, no period at end
- **Body**: Wrapped at 72 characters
- **Signed-off-by**: Required (Developer Certificate of Origin)
- **Structure**: Problem/Solution/Limitations sections for non-trivial changes

### Examples

```
Fix deadlock in ncclAllReduce with multi-node InfiniBand

Problem: When using IB transport with >8 nodes, ncclAllReduce
could deadlock if the ring topology crossed NUMA boundaries due
to incorrect buffer registration ordering.

Solution: Register IB buffers in rank order rather than channel
order, ensuring consistent ordering across all nodes.

Limitations: Does not address the similar issue in ncclAllGather,
which will be fixed in a follow-up commit.

Signed-off-by: Developer Name <dev@example.com>
```

<!-- source: Codex inference development article, SGLang development article -->

# Torch Compile Coverage for Serving Paths

## What This Covers

This note captures a recurring inference-framework failure mode:

- a custom kernel exists
- the kernel is fast in isolation
- end-to-end serving still regresses because the path breaks `torch.compile`

Use this document when the question is not only "is this kernel fast?" but also:

- does this op stay inside a compile region?
- did a custom op or wrapper increase graph breaks?
- is the serving stack actually using the optimized path end to end?

## Why This Matters

Serving performance is often decided by graph shape, not by one kernel benchmark alone.

A kernel swap can regress the system when:

- the new path is not registered as a proper custom op
- wrapper code creates graph breaks
- the optimized op is excluded from the compile region

In those cases, an apparently faster kernel can still make the model slower.

## Default Debug Loop

1. benchmark the candidate op on representative shapes
1. integrate it into the real serving or model path
1. measure end-to-end again
1. if performance regresses or does not improve, inspect graph coverage before touching the kernel again

Do not assume an operator-level speedup will survive integration.

## Primary Check: Graph Coverage

Use `torch._dynamo.explain` or an equivalent graph-break inspection tool to answer:

- how many graph breaks exist?
- which call boundary caused the break?
- is the target kernel or wrapper inside the compiled region?

The first practical question is:

- "is the hot path covered by `torch.compile` or not?"

## Common Root Causes

| Symptom | Likely Cause | What to Check |
| ------- | ------------ | ------------- |
| operator benchmark is faster but E2E is slower | graph break increased | inspect compile regions before and after integration |
| custom kernel never appears inside the graph | missing or incomplete custom-op registration | inspect registration path and wrapper boundary |
| one hot wrapper blocks multiple downstream ops | wrapper function is opaque to compile | identify whether a thinner wrapper or registered op fixes coverage |
| profiler still shows old fallback path | compile excluded the new path | compare graph coverage and trace output together |

## Registration Guidance

When integrating a custom kernel into a serving framework:

- register the op cleanly instead of leaving it as an opaque Python call when possible
- keep wrapper logic thin
- avoid mixing unrelated control logic into the hot path
- re-check graph coverage after every integration change

The goal is not only to expose the kernel, but to expose it in a form the compile stack
can consume.

## Evidence To Preserve

For any compile-coverage optimization, keep:

- representative input shape
- operator benchmark result
- end-to-end benchmark result
- `torch._dynamo.explain` output or equivalent graph-break report
- profiler evidence showing whether the hot path entered the compile region

Without all five, it is too easy to over-attribute the result to the kernel itself.

## Recommended Interpretation Order

1. check end-to-end regression or improvement
1. inspect graph-break count and location
1. confirm whether the target op is inside the compile region
1. only then decide whether to change registration, wrapper shape, or the kernel itself

This order prevents low-level churn when the real issue is integration.

## Review Questions

1. was the op tested both in isolation and inside the real serving path?
1. was graph coverage checked after integration?
1. is the custom op registered in a compile-friendly way?
1. does profiler evidence confirm the optimized path entered the compile region?

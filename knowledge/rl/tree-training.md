<!-- source: AReaL -->

# Tree Training and Prefix Sharing

## What It Is

Tree training is a sequence packing strategy for RL and agent workloads where many
samples share the same prompt prefix. Instead of recomputing attention for each sequence
independently, shared prefixes are represented once in a tree or trie and divergent
suffixes branch from that shared path.

This is most valuable when:

- `n_samples > 1` for the same prompt
- multi-turn conversations share long history prefixes
- prompts share large system prompts or few-shot context

The payoff can be large because prefix tokens dominate FLOPs in long-context RL.

## Mental Model

```text
Seq0: [A, B, C, D]
Seq1: [A, B, E, F]
Seq2: [A, G, H]

Packed tree:
    [A]
    / \
  [B] [G]
  / \   \
[C] [E] [H]
 |   |
[D] [F]
```

The system computes the shared prefix once and reuses it across branches.

## Where It Helps Most

### Agentic RL

Agent workloads naturally create prefix sharing:

- repeated sampling from the same user prompt
- retries after tool failure
- branch exploration from the same conversation state
- best-of-n or tree search style rollouts

### Multi-turn conversation training

When training on a running conversation, most turns reuse the full history prefix. Tree
packing prevents paying full attention cost for that history on every branch.

## Generic Pipeline

1. extract the real token sequence for each sample
1. group sequences that can share prefixes
1. insert them into a trie or similar prefix structure
1. compress linear chains into multi-token nodes
1. emit an attention representation usable by the chosen backend
1. reconstruct per-sequence logprobs and metrics from the packed representation

## Implementation Patterns

### Trie or compressed-tree representation

A useful node shape tracks:

- token span stored at this node
- child branches
- ancestors
- sequence IDs traversing the node
- flattened start and end offsets for packed execution

Compression matters. A naive token-per-node trie adds too much overhead. Merge linear
chains so the structure remains cheap enough to justify the packing.

### Backend-specific attention wrapper

Do not leak tree internals into every model implementation. Instead, build a narrow
adapter layer that converts packed tree metadata into whatever the attention backend
needs:

- sparse block masks
- dense masks for checkpointing-constrained paths
- custom Triton metadata
- model-specific tree attention wrappers

### Per-sequence logprob reconstruction

Packed execution changes token positions, so simple label shifting is not enough. The
runtime must reconstruct sequence-local logprobs from node-local computations and cache
shared prefix work where possible.

## Configuration Invariants

Tree training usually requires stricter invariants than ordinary batching.

Typical requirements:

- padding to a known maximum length
- explicit max tokens per microbatch or per packed tree
- block-size alignment for sparse attention kernels

If these are not validated early, failures show up as opaque runtime shape errors.

## Metrics Worth Tracking

Track how much real sharing you are getting. A simple generic metric is:

- `tree_token_ratio`: packed token count divided by original token count

Interpretation:

- close to `1.0`: little sharing, complexity may not be worth it
- meaningfully below `1.0`: packing is saving real compute

Also watch:

- tokens per packed tree
- average branch factor
- reconstruction overhead
- end-to-end step time, not just model FLOPs

## Constraints and Tradeoffs

Tree training is powerful, but it is not free.

Common tradeoffs:

- some parallel modes are incompatible with packed attention paths
- MoE models may become numerically less stable under alternative attention kernels
- checkpointing paths may require dense tensors instead of richer sparse objects
- implementation complexity is much higher than ordinary batching

Use it when prefix sharing is large enough to dominate that complexity.

## Review Questions

Before adopting a tree-training design, ask:

1. do our workloads actually have large repeated prefixes, or are we optimizing the wrong bottleneck?
1. is the chosen attention backend compatible with our checkpointing and parallel strategy?
1. do we have a correct path to reconstruct per-sequence logprobs and rewards?
1. can we disable the feature cleanly when numerics or debugging require a simpler path?
1. are we measuring end-to-end training throughput rather than only theoretical FLOP savings?

## Good Placement in an AI Infra Repo

Tree training belongs in `knowledge`, not only in a workflow guide, when:

- the idea spans multiple engines
- the constraints are backend-specific
- reviewers need a reference for packed attention tradeoffs
- the design will likely be reused in more than one project

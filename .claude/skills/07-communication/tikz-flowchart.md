# TikZ Flowchart

Use this guide when the user wants a professional technical diagram in LaTeX TikZ.

## Best For

- algorithm flowcharts
- model or training architecture diagrams
- kernel execution or data-flow diagrams
- clean PDF-ready technical visuals

## Style Defaults

- green nodes for data or tensors
- orange or memory-toned nodes for weights, checkpoints, or storage
- blue nodes for compute or operations
- dashed grouping boxes for phases or subsystems
- orthogonal edges and relative positioning for readability

## Workflow

1. identify the minimal set of entities and transitions
1. choose stable node categories before drawing
1. group related phases with background containers
1. keep text short and use line breaks instead of dense paragraphs
1. prefer orthogonal edges over diagonal clutter

## Review Rules

- diagrams should reflect the real system contract
- avoid mixing implementation detail and audience-level concepts in one figure
- if the diagram explains a kernel or distributed workflow, align the naming with the code

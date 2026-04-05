# Skill Authoring Guide

This document captures the repo-level conventions for writing and evolving skills in
`ai-infra-skills`.

It exists to keep the skill system small, composable, and maintainable as the repository
 grows.

## Core Lessons

The most useful skills are not giant prompts. They are small, specific task modules that
load the right context at the right time.

In practice, that means:

- keep one skill focused on one job family
- prefer composition over mega-skills
- keep category `SKILL.md` files as routers, not encyclopedias
- move deep background into `knowledge/`
- move judgment-heavy domain routing into experts

If a skill tries to teach the entire domain, it will become bloated and harder for the
agent to activate reliably.

## The Five Building Blocks in This Repo

### 1. `skills`

Use a skill for executable workflow guidance.

Good fit:

- add a workflow
- debug a CUDA crash
- triage a CI failure
- benchmark a kernel

Bad fit:

- durable domain theory
- long reference material
- agent personality or expert boundaries

### 2. `knowledge`

Use `knowledge/` for deep reference material that supports experts and skills.

Good fit:

- NCCL patterns
- tree training tradeoffs
- gateway serving contracts
- model onboarding debug loops

### 3. `agents`

Use an expert agent when a topic needs cross-repo judgment and repeated routing.

Good fit:

- FSDP vs Megatron choice
- kernel-level diagnosis
- CI failure bucketing
- serving-topology tradeoffs

### 4. `rules`

Use rules for constraints that should persist across tasks.

Good fit:

- API compatibility
- testing discipline
- distributed invariants

### 5. `commands`

Use commands for direct user-invoked action flows.

Good fit:

- commit generation
- PR review
- create PR
- bisect

## Skill Taxonomy for This Repo

We use three practical skill shapes:

### Category Router

These are the `SKILL.md` files inside each numbered skill directory.

Purpose:

- help the agent choose the right guide
- define scope boundaries for the category
- keep top-level context small

Rule:

- category routers should summarize, not duplicate all detail from child guides

### Task Workflow Guide

These are the task-specific docs under each skill directory.

Purpose:

- describe one workflow end to end
- provide steps, checks, failure modes, and verification

Rule:

- a workflow guide should be narrow enough that activation is obvious

### Cross-Linking Support

A skill should point out:

- which expert to consult when the task becomes judgment-heavy
- which knowledge docs explain the hard parts
- which rules constrain the final implementation

Rule:

- skills should not silently absorb adjacent domains when a cross-link is clearer

## Progressive Disclosure

Use progressive disclosure to keep context tight:

1. category `SKILL.md` explains when to use the category
1. task guide explains how to execute one workflow
1. `knowledge/` explains why the hard parts work that way
1. expert agent helps choose between options when the answer is not mechanical

If a single file needs all four layers at once, the design is probably too large.

## Skill Template Expectations

Every new workflow guide should normally include:

- what it is for
- when to activate
- when not to use it
- step-by-step process
- verification or review checklist
- common mistakes or gotchas

Every category `SKILL.md` should normally include:

- category summary
- included guides
- general principles for the category

## Gotchas Are First-Class

One of the highest-leverage patterns is explicitly recording where the agent is likely to
fail.

Add a gotcha or common-mistakes section when:

- the same failure mode appears across multiple repos
- the wrong default action is tempting
- a task has one or two subtle invariants that break everything

Examples:

- using a skill when a rule should own the behavior
- mixing serving semantics with RL semantics
- debugging a distributed hang before checking rank layout
- tuning PPO without logging reward and ratio statistics first

## Quality Bar for New Skills

Before adding a new skill, check:

1. is this truly a repeatable workflow instead of one project note?
1. is the scope narrow enough that activation will be reliable?
1. can most background detail live in `knowledge/` instead?
1. are there clear adjacent experts or rules to link?
1. does the skill encode at least one non-obvious gotcha?

## Maintenance Rules

Treat skills like production assets, not static prompts.

Recommended maintenance behavior:

- revise skills after real failures, not only after successful use
- keep examples representative of real tasks
- version behavior through normal git history instead of silent prompt drift
- prefer measured utility over "seems good"

When a skill becomes too large:

- split the workflow into smaller guides
- move theory into `knowledge/`
- move routing judgment into an expert

## Repo-Specific Defaults

For `ai-infra-skills`, default to:

- small composable workflow guides
- expert-owned knowledge instead of hidden background inside skills
- gotcha-first writing for distributed, kernel, serving, and RL topics
- category routers that stay short enough to be loaded often

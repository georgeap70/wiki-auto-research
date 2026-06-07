---
title: The Self-Improvement Loop
type: concept
tags: [core-concept, loop, measure-fail-propose-gate]
sources: [agent0, auto-harness, autoresearch-vs-hpo, meta-harness, optimize-anything, neosigma-blog, evox, autoagent, autoagent2, asi-evolve, coral, deep-research, agentflow, trace, autogenesis, webxskill, halo, autoreason, skillOpt, evo-hq]
last_updated: 2026-06-06
---

# The Self-Improvement Loop

The fundamental architecture underlying all self-improving agentic systems in this wiki. Every system instantiates a version of:

```
measure → fail → propose → gate → repeat
```

## The Four Phases

### 1. Measure
Run the agent on a benchmark, production workload, or eval suite. Collect performance data.

### 2. Fail
Identify failures. The quality of failure analysis is a key differentiator:
- **Shallow**: count of failed tasks (scalar)
- **Rich**: cluster failures by root cause, extract execution traces, identify specific lines of code or prompts responsible

See [[concepts/feedback-signals]] for the argument that rich failure analysis dramatically improves the next phase.

### 3. Propose
Generate a candidate improvement. This is the creative step. What is proposed depends on what level the system operates at:

| Level | Proposal type | Example system |
|-------|--------------|----------------|
| Task solutions | Better answer or reasoning chain | [[sources/agent0]] |
| Constraint code | Harness that prevents invalid actions | [[sources/autoharness-arxiv]] |
| System prompts | Modified agent prompts | [[sources/auto-harness]], [[sources/meta-harness]], [[sources/autoagent-kevinrgu]], [[sources/deep-research]] |
| Training config → architecture | Code changes to training loop | [[sources/autoresearch-vs-hpo]] |
| Agent architecture + RL algorithm | Structural code growth + learning algorithm | [[sources/optimize-anything]], [[sources/asi-evolve]] |
| Code solutions (open-ended) | Any code that maximizes grader score | [[sources/coral]] |
| Data curation pipelines | Preprocessing strategies for training corpora | [[sources/asi-evolve]] |
| Per-capability LoRA adapters | New modular weight deltas + router | [[sources/trace]] |
| Executable skills (param program + NL) | New dual-mode skill artifacts | [[sources/webxskill]] |
| Typed versioned resources (prompts/tools/memory) | Protocol-level modifications with rationale | [[sources/autogenesis]] |
| The optimization algorithm | Switch search strategy | [[sources/evox]] |

### 4. Gate
Accept or reject the proposed change. This is the safety mechanism:
- **Threshold gating**: accept only if ≥80% of prior tasks still pass ([[sources/auto-harness]])
- **Pareto gating**: accept if non-dominated on a multi-metric frontier ([[sources/optimize-anything]])
- **Held-out eval**: test on tasks not seen during proposal ([[sources/meta-harness]])
- **Stagnation detection**: accept strategy switch only if current strategy has plateaued ([[sources/evox]])
- **Novelty filtering**: reject candidates whose motivation is near-duplicate of existing ones ([[sources/asi-evolve]])
- **Heartbeat pivoting**: force algorithmic pivot after N consecutive non-improving evaluations ([[sources/coral]])
- **Lineage + rollback**: accept changes but keep them reversible via versioned resources with decision rationale ([[sources/autogenesis]])
- **Inheritable tree gates**: pass/fail checks attached to nodes in the experiment tree; a root gate runs on every descendant; gate failure dominates score improvement ([[sources/evo]])

See [[concepts/regression-gating]] for details.

## Loop Architectures

The loop can be instantiated in several ways:

### Single-agent self-edit
One agent edits its own code, prompts, or config. Simple. Used by [[sources/auto-harness]], [[sources/meta-harness]].

### Two-agent co-evolution
Two agents (curriculum + executor) each improve by competing with the other. Neither needs external training data. Used by [[sources/agent0]].

### Population-based evolution
A population of candidate solutions/architectures evolves under selection pressure. Used by [[sources/optimize-anything]], [[sources/evox]], [[sources/coral]].

### Multi-agent co-evolution with shared memory
Multiple autonomous agents explore in parallel, each running the full loop, sharing a persistent knowledge store. No external algorithm controls which candidates are retrieved or evaluated — agents direct themselves. Used by [[sources/coral]].

Emergent coordination arises from shared memory access alone: agents copy successful peer techniques (copycatting), synthesize patterns across notes (cross-referencing), and form consensus on exhausted directions. See [[concepts/knowledge-accumulation]].

### Learn–Design–Experiment–Analyze
A four-stage variant used by [[sources/asi-evolve]] for long-horizon scientific research loops:
- **Learn:** retrieve relevant prior work and past discoveries from a [[concepts/knowledge-accumulation|Cognition Base]]
- **Design (Researcher agent):** generate candidate + motivation
- **Experiment (Engineer agent):** run evaluation with safeguards
- **Analyze (Analyzer agent):** distill multi-dimensional experimental output into compact, decision-oriented report; store node in database

The Analyzer stage compresses feedback complexity — preventing context explosion in domains with rich, noisy experimental signals.

### In-the-flow on-policy training
The agent's policy weights are updated **during live task execution** via RL, not between task batches. There is no separate improvement phase. Used by [[sources/agentflow]].

The key mechanism is **Flow-GRPO**: a trajectory-level binary outcome signal is broadcast to every intermediate decision step, converting multi-turn credit assignment into tractable single-turn policy updates. The planner module improves continuously while the system is running.

### Capability-decomposed training
A semi-autonomous variant used by [[sources/trace]]:
- **Analysis agent** contrasts successful vs. failed trajectories to diagnose capability gaps
- **Generation agent** synthesizes per-capability training environments with dense isolated rewards
- Lightweight **LoRA adapters** are trained one per capability via GRPO
- A **router** composes them at inference

The agent being improved does not drive its own diagnosis — supervising LLM agents do. This is less autonomous than [[sources/coral]] or [[sources/agent0]] but the capability-decomposition mechanism is orthogonal and could be composed with a fully autonomous diagnostic stage.

### Per-query tournament refinement
[[sources/autoreason]] is a different *scope* of loop entirely: rather than improving a system across queries, it improves a single output across iterations *within* one query. Each iteration produces three candidates (incumbent, adversarial revision, synthesis) and a blind judge panel selects the winner via Borda count. Convergence: incumbent wins twice. Notable for explicitly addressing three pathologies of naive critique-revise loops (prompt bias, scope creep, lack of restraint) by replacing critique with comparison.

### Production-trace deployment loop
[[sources/halo]] explicitly designs the loop around live deployment traffic: deploy → collect OpenTelemetry traces from real users → specialized RLM analyzes cross-trace patterns → coding agent edits harness → redeploy. The RLM's role is to compress many production traces into actionable systemic-failure reports. This is the same architecture as [[sources/auto-harness]]'s overnight loop, formalized as a methodology with separated trace-analysis and harness-editing components.

### Skill-document gradient descent
[[sources/skillopt]] frames the entire loop as gradient descent on a *text artifact*. The skill document `best_skill.md` plays the role of model weights; the optimizer model plays the role of the gradient computation; the edit budget plays the role of the learning rate; the validation gate plays the role of accept/reject on each step. Reflection over batched successes and failures plays the role of the loss-and-gradient computation. The result is a self-improvement loop that is structurally near-identical to SGD but operates at the prose-document layer with no weight changes.

### Tree-structured parallel hill-climb with cross-cutting scans
[[sources/evo]] sits between single-thread hill-climb and flat population evolution. The orchestrator maintains a *tree of committed experiments*. Each round, a configurable **frontier strategy** (`argmax`, `top_k`, `epsilon_greedy`, `softmax`, `pareto_per_task`) picks which committed branch to extend; within the round, parallel subagents in isolated git worktrees each pick up shared state (failure traces, annotations, discarded hypotheses), form a hypothesis, edit, and benchmark. Between rounds, RLM-inspired **cross-cutting scan subagents** read trace batches in parallel and surface compound failure patterns (gate-failure intersections, shared root causes); their findings land in shared state for the next round.

Two distinguishing features: (a) tree shape preserves lineage and exposes the selection operator as a tunable knob (the `pareto_per_task` strategy is credited to GEPA); (b) cross-cutting scans are a *standing between-round phase*, not a one-shot analyzer — feedback compression woven into the loop rather than bolted on at the end.

### Protocol-governed self-modification
[[sources/autogenesis]] formalizes the loop as a **protocol** rather than an algorithm. The measure → fail → propose → gate → repeat cycle is implemented as typed operators (Proposal / Assessment / Commitment) over versioned resources (prompts, tools, agents, environments, memory). Every commitment leaves a rollback path; every modification carries decision rationale.

This is not a new loop *shape* — it's a specification of what primitives a self-improving system needs to expose for external inspection, audit, and safe reversal.

### Together: weight optimization in this wiki

This is the set of loop types that optimize **weights** rather than harness code, prompts, or architecture:

| System | What weights are trained | How |
|--------|--------------------------|-----|
| [[sources/agentflow]] | Planner module weights | Flow-GRPO, in-the-flow |
| [[sources/skill-rl-skill0]] SKILL-RL | Base policy weights | RL with external SkillBank in context |
| [[sources/skill-rl-skill0]] SKILL-0 | Base policy weights | Curriculum RL, progressive skill withdrawal |
| [[sources/trace]] | Per-capability LoRA adapters | GRPO in capability-isolated environments |

### Meta-evolution
The evolution strategy itself is subject to the same evolutionary loop. Used by [[sources/evox]].

## What Makes the Loop Compound?

Without compounding, each iteration starts fresh. What creates compounding:
- **Persistent memory** (`learnings.md` in [[sources/auto-harness]]; Attempts/Notes/Skills in [[sources/coral]]; Cognition Base in [[sources/asi-evolve]])
- **Retained population** (evolutionary approaches maintain a Pareto frontier)
- **Curriculum escalation** (harder tasks only appear once easier ones are mastered — [[sources/agent0]])
- **Structural growth** (the agent's code grows and becomes more capable over time — [[sources/optimize-anything]])
- **Shared knowledge distillation** (successful techniques are extracted into reusable Skills — [[sources/coral]])

See [[concepts/knowledge-accumulation]] for a detailed treatment of how different systems implement persistence.

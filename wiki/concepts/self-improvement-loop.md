---
title: The Self-Improvement Loop
type: concept
tags: [core-concept, loop, measure-fail-propose-gate]
sources: [agent0, auto-harness, autoresearch-vs-hpo, meta-harness, optimize-anything, optimize-anything-omni, neosigma-blog, evox, autoagent, autoagent2, asi-evolve, coral, deep-research, agentflow, trace, autogenesis, webxskill, halo, autoreason, skillOpt, evo-hq, ophis, self-evolving]
last_updated: 2026-07-31
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

See [concepts/feedback-signals](feedback-signals.md) for the argument that rich failure analysis dramatically improves the next phase.

### 3. Propose
Generate a candidate improvement. This is the creative step. What is proposed depends on what level the system operates at:

| Level | Proposal type | Example system |
|-------|--------------|----------------|
| Task solutions | Better answer or reasoning chain | [sources/agent0](../sources/agent0.md) |
| Constraint code | Harness that prevents invalid actions | [sources/autoharness-arxiv](../sources/autoharness-arxiv.md) |
| System prompts | Modified agent prompts | [sources/auto-harness](../sources/auto-harness.md), [sources/meta-harness](../sources/meta-harness.md), [sources/autoagent-kevinrgu](../sources/autoagent-kevinrgu.md), [sources/deep-research](../sources/deep-research.md) |
| Training config → architecture | Code changes to training loop | [sources/autoresearch-vs-hpo](../sources/autoresearch-vs-hpo.md) |
| Agent architecture + RL algorithm | Structural code growth + learning algorithm | [sources/optimize-anything](../sources/optimize-anything.md), [sources/asi-evolve](../sources/asi-evolve.md) |
| Code solutions (open-ended) | Any code that maximizes grader score | [sources/coral](../sources/coral.md) |
| Data curation pipelines | Preprocessing strategies for training corpora | [sources/asi-evolve](../sources/asi-evolve.md) |
| Per-capability LoRA adapters | New modular weight deltas + router | [sources/trace](../sources/trace.md) |
| Executable skills (param program + NL) | New dual-mode skill artifacts | [sources/webxskill](../sources/webxskill.md) |
| Typed versioned resources (prompts/tools/memory) | Protocol-level modifications with rationale | [sources/autogenesis](../sources/autogenesis.md) |
| The optimization algorithm | Switch search strategy | [sources/evox](../sources/evox.md) |

### 4. Gate
Accept or reject the proposed change. This is the safety mechanism:
- **Threshold gating**: accept only if ≥80% of prior tasks still pass ([sources/auto-harness](../sources/auto-harness.md))
- **Pareto gating**: accept if non-dominated on a multi-metric frontier ([sources/optimize-anything](../sources/optimize-anything.md))
- **Held-out eval**: test on tasks not seen during proposal ([sources/meta-harness](../sources/meta-harness.md))
- **Stagnation detection**: accept strategy switch only if current strategy has plateaued ([sources/evox](../sources/evox.md))
- **Novelty filtering**: reject candidates whose motivation is near-duplicate of existing ones ([sources/asi-evolve](../sources/asi-evolve.md))
- **Heartbeat pivoting**: force algorithmic pivot after N consecutive non-improving evaluations ([sources/coral](../sources/coral.md))
- **Lineage + rollback**: accept changes but keep them reversible via versioned resources with decision rationale ([sources/autogenesis](../sources/autogenesis.md))
- **Inheritable tree gates**: pass/fail checks attached to nodes in the experiment tree; a root gate runs on every descendant; gate failure dominates score improvement ([sources/evo](../sources/evo.md))

See [concepts/regression-gating](regression-gating.md) for details.

## Loop Architectures

The list below organizes loops by their **mechanism**. An orthogonal cut — *where the change lands and how long it persists* — comes from [sources/self-evolving](../sources/self-evolving.md)'s What×When taxonomy: any loop can be indexed by substrate (external files / harness / weights) and horizon (single session / across sessions / across users). The "weight optimization" subsection below is exactly the *weights* column; most other loops here live in the *harness across-sessions* cell.

The loop can be instantiated in several ways:

### Single-agent self-edit
One agent edits its own code, prompts, or config. Simple. Used by [sources/auto-harness](../sources/auto-harness.md), [sources/meta-harness](../sources/meta-harness.md).

### Two-agent co-evolution
Two agents (curriculum + executor) each improve by competing with the other. Neither needs external training data. Used by [sources/agent0](../sources/agent0.md).

### Population-based evolution
A population of candidate solutions/architectures evolves under selection pressure. Used by [sources/optimize-anything](../sources/optimize-anything.md), [sources/evox](../sources/evox.md), [sources/coral](../sources/coral.md).

### Multi-agent co-evolution with shared memory
Multiple autonomous agents explore in parallel, each running the full loop, sharing a persistent knowledge store. No external algorithm controls which candidates are retrieved or evaluated — agents direct themselves. Used by [sources/coral](../sources/coral.md).

Emergent coordination arises from shared memory access alone: agents copy successful peer techniques (copycatting), synthesize patterns across notes (cross-referencing), and form consensus on exhausted directions. See [concepts/knowledge-accumulation](knowledge-accumulation.md).

### Learn–Design–Experiment–Analyze
A four-stage variant used by [sources/asi-evolve](../sources/asi-evolve.md) for long-horizon scientific research loops:
- **Learn:** retrieve relevant prior work and past discoveries from a [Cognition Base](knowledge-accumulation.md)
- **Design (Researcher agent):** generate candidate + motivation
- **Experiment (Engineer agent):** run evaluation with safeguards
- **Analyze (Analyzer agent):** distill multi-dimensional experimental output into compact, decision-oriented report; store node in database

The Analyzer stage compresses feedback complexity — preventing context explosion in domains with rich, noisy experimental signals.

### In-the-flow on-policy training
The agent's policy weights are updated **during live task execution** via RL, not between task batches. There is no separate improvement phase. Used by [sources/agentflow](../sources/agentflow.md).

The key mechanism is **Flow-GRPO**: a trajectory-level binary outcome signal is broadcast to every intermediate decision step, converting multi-turn credit assignment into tractable single-turn policy updates. The planner module improves continuously while the system is running.

### Capability-decomposed training
A semi-autonomous variant used by [sources/trace](../sources/trace.md):
- **Analysis agent** contrasts successful vs. failed trajectories to diagnose capability gaps
- **Generation agent** synthesizes per-capability training environments with dense isolated rewards
- Lightweight **LoRA adapters** are trained one per capability via GRPO
- A **router** composes them at inference

The agent being improved does not drive its own diagnosis — supervising LLM agents do. This is less autonomous than [sources/coral](../sources/coral.md) or [sources/agent0](../sources/agent0.md) but the capability-decomposition mechanism is orthogonal and could be composed with a fully autonomous diagnostic stage.

### Per-query tournament refinement
[sources/autoreason](../sources/autoreason.md) is a different *scope* of loop entirely: rather than improving a system across queries, it improves a single output across iterations *within* one query. Each iteration produces three candidates (incumbent, adversarial revision, synthesis) and a blind judge panel selects the winner via Borda count. Convergence: incumbent wins twice. Notable for explicitly addressing three pathologies of naive critique-revise loops (prompt bias, scope creep, lack of restraint) by replacing critique with comparison.

### Production-trace deployment loop
[sources/halo](../sources/halo.md) explicitly designs the loop around live deployment traffic: deploy → collect OpenTelemetry traces from real users → specialized RLM analyzes cross-trace patterns → coding agent edits harness → redeploy. The RLM's role is to compress many production traces into actionable systemic-failure reports. This is the same architecture as [sources/auto-harness](../sources/auto-harness.md)'s overnight loop, formalized as a methodology with separated trace-analysis and harness-editing components.

### Skill-document gradient descent
[sources/skillopt](../sources/skillopt.md) frames the entire loop as gradient descent on a *text artifact*. The skill document `best_skill.md` plays the role of model weights; the optimizer model plays the role of the gradient computation; the edit budget plays the role of the learning rate; the validation gate plays the role of accept/reject on each step. Reflection over batched successes and failures plays the role of the loss-and-gradient computation. The result is a self-improvement loop that is structurally near-identical to SGD but operates at the prose-document layer with no weight changes.

### Tree-structured parallel hill-climb with cross-cutting scans
[sources/evo](../sources/evo.md) sits between single-thread hill-climb and flat population evolution. The orchestrator maintains a *tree of committed experiments*. Each round, a configurable **frontier strategy** (`argmax`, `top_k`, `epsilon_greedy`, `softmax`, `pareto_per_task`) picks which committed branch to extend; within the round, parallel subagents in isolated git worktrees each pick up shared state (failure traces, annotations, discarded hypotheses), form a hypothesis, edit, and benchmark. Between rounds, RLM-inspired **cross-cutting scan subagents** read trace batches in parallel and surface compound failure patterns (gate-failure intersections, shared root causes); their findings land in shared state for the next round.

Two distinguishing features: (a) tree shape preserves lineage and exposes the selection operator as a tunable knob (the `pareto_per_task` strategy is credited to GEPA); (b) cross-cutting scans are a *standing between-round phase*, not a one-shot analyzer — feedback compression woven into the loop rather than bolted on at the end.

### Mechanistic hypothesis-driven loop (non-search)
[sources/ophis](../sources/ophis.md) is the wiki's one loop that is **neither LLM-proposer-driven nor evolutionary**. Its cycle is **Observation → Problem → Hypothesis → Intervention → Speed-up**: measure ~6,000 tensor-level training-dynamics observables, localize a bottleneck, form a *causal* hypothesis about why it occurs, derive a targeted intervention *from that hypothesis*, and validate the speed-up. The "propose" step is deduction from a mechanistic model, not sampling from a proposer or a population — which is why it can generate candidates "almost instantly" and fail far less often (13.7% vs 42.1% for an LLM baseline on grokking). Gating is by *mechanistic plausibility* plus a mean/stability/variance criterion, with 10 repeated evals to beat kernel noise.

> OPHIS also carries its own **Stage 1/2/3 taxonomy of causal depth** (internet-prior recombination → statistical memory/RSI → mechanistic reasoning). This is orthogonal to — and easily confused with — [CORAL's Stage 1/2/3 *autonomy* taxonomy](../sources/coral.md). They rank different axes; see [sources/ophis](../sources/ophis.md).

### Protocol-governed self-modification
[sources/autogenesis](../sources/autogenesis.md) formalizes the loop as a **protocol** rather than an algorithm. The measure → fail → propose → gate → repeat cycle is implemented as typed operators (Proposal / Assessment / Commitment) over versioned resources (prompts, tools, agents, environments, memory). Every commitment leaves a rollback path; every modification carries decision rationale.

This is not a new loop *shape* — it's a specification of what primitives a self-improving system needs to expose for external inspection, audit, and safe reversal.

### Together: weight optimization in this wiki

This is the set of loop types that optimize **weights** rather than harness code, prompts, or architecture:

| System | What weights are trained | How |
|--------|--------------------------|-----|
| [sources/agentflow](../sources/agentflow.md) | Planner module weights | Flow-GRPO, in-the-flow |
| [sources/skill-rl-skill0](../sources/skill-rl-skill0.md) SKILL-RL | Base policy weights | RL with external SkillBank in context |
| [sources/skill-rl-skill0](../sources/skill-rl-skill0.md) SKILL-0 | Base policy weights | Curriculum RL, progressive skill withdrawal |
| [sources/trace](../sources/trace.md) | Per-capability LoRA adapters | GRPO in capability-isolated environments |

### Meta-evolution
The evolution strategy itself is subject to the same evolutionary loop. Used by [sources/evox](../sources/evox.md).

## What Makes the Loop Compound?

Without compounding, each iteration starts fresh. What creates compounding:
- **Persistent memory** (`learnings.md` in [sources/auto-harness](../sources/auto-harness.md); Attempts/Notes/Skills in [sources/coral](../sources/coral.md); Cognition Base in [sources/asi-evolve](../sources/asi-evolve.md))
- **Retained population** (evolutionary approaches maintain a Pareto frontier)
- **Curriculum escalation** (harder tasks only appear once easier ones are mastered — [sources/agent0](../sources/agent0.md))
- **Structural growth** (the agent's code grows and becomes more capable over time — [sources/optimize-anything](../sources/optimize-anything.md))
- **Shared knowledge distillation** (successful techniques are extracted into reusable Skills — [sources/coral](../sources/coral.md))

See [concepts/knowledge-accumulation](knowledge-accumulation.md) for a detailed treatment of how different systems implement persistence.

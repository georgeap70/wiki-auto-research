---
title: Knowledge Accumulation in Self-Improving Systems
type: concept
tags: [knowledge, memory, persistence, compounding, cognition-base, shared-memory, adapters, protocol-lineage]
sources: [auto-harness, meta-harness, asi-evolve, coral, skill0, webxskill, trace, autogenesis, skillOpt, rlm_gepa, self-evolving]
last_updated: 2026-07-31
---

# Knowledge Accumulation

A recurring theme across self-improving systems: **persistent memory that compounds across iterations** is what separates genuine improvement from random search. Without it, each iteration starts from scratch; with it, discoveries from earlier rounds inform later ones.

## Why It Matters

The naive self-improvement loop has no memory. The agent proposes a change, the change is accepted or rejected, and the next proposal starts from the same base. This is expensive: the same failure modes are rediscovered repeatedly, and insights from failed attempts are discarded.

Knowledge accumulation solves this by storing structured information that future iterations can retrieve. The form of storage varies by system, but the function is constant: **convert experience into reusable context**.

## Implementations Across Sources

### learnings.md — [sources/auto-harness](../sources/auto-harness.md)

A flat markdown file maintained alongside the agent harness. Each improvement session appends findings: what worked, what failed, which failure clusters have been seen before. On next session, this file is injected into the optimizer's context.

Simple, but effective for sequential single-agent loops. Fails to scale across very long runs or parallel agents.

### Execution Trace Database — [sources/meta-harness](../sources/meta-harness.md)

Stanford's Meta-Harness stores full execution traces (up to 10M tokens). The optimizer can follow the execution path of a failure to its source. More expensive than learnings.md, but richer — the trace contains the actual causal chain, not just a summary.

### Cognition Base — [sources/asi-evolve](../sources/asi-evolve.md)

An embedding-indexed repository of human prior knowledge (papers, heuristics, known pitfalls) plus agent-generated analysis nodes. Two-layer structure:

1. **Human priors:** seeded from domain literature (e.g., 150 entries from 100 architecture papers)
2. **Agent analyses:** each Analyzer report is stored as a node with motivation, code, results, and analysis

Retrieved via semantic search over candidate motivations. This enables the agent to:
- Leverage existing human expertise without re-deriving it
- Build on its own prior discoveries without redundant exploration (novelty check filters near-duplicate motivations)

### Attempts / Notes / Skills — [sources/coral](../sources/coral.md)

The richest accumulation architecture seen in this literature. Three parallel stores:

**Attempts:** JSON per evaluated commit. Complete historical record — agent ID, score, parent hash, grader feedback. Agents can inspect the full lineage of any solution.

**Notes:** Freeform markdown organized by topic. Special directories: `_synthesis/` for consolidated findings, `_connections.md` for cross-category patterns, `_open-questions.md` for gaps. Agents write, read, and *reorganize* notes — the structure of knowledge evolves alongside the content.

**Skills:** Reusable procedures with natural-language description + executable artifacts. Once an agent discovers a useful technique, it distills it into a skill that any colleague can invoke.

The three layers serve different time horizons: Attempts are real-time (per evaluation), Notes are medium-term (per heartbeat), Skills are long-term (validated across many runs).

### SkillBank — [sources/skill-rl-skill0](../sources/skill-rl-skill0.md) (SKILL-RL)

SKILL-RL extracts reusable **skills** from RL rollouts — not raw trajectories but distilled behavioral patterns with natural-language descriptions + executable artifacts. Organized hierarchically:

- **General heuristics** at the top (broadly applicable across task types)
- **Task-specific procedures** below (retrieved when task context matches)

Crucially, the SkillBank **co-evolves with the RL policy**: as the policy improves through training, the SkillBank is updated to reflect the current best practices. The loop is recursive — better skills produce better rollouts, which produce better skills.

This is the closest analog in the literature to CORAL's Skills store, but tightly coupled to RL training rather than agent co-evolution.

### Weight Internalization — [sources/skill-rl-skill0](../sources/skill-rl-skill0.md) (SKILL-0)

All prior knowledge accumulation systems are **external** (files, databases, embeddings, git). SKILL-0 is the only system in this wiki that stores accumulated knowledge **in model weights**.

The mechanism is a dynamic curriculum:
1. Training begins with full skill context in-context
2. The skill budget is linearly decayed across training steps
3. The model is trained to produce the same correct behavior with progressively less external scaffolding
4. By the end of training, the model operates zero-shot — skills have been internalized

This eliminates inference-time retrieval entirely (< 0.5k tokens/step vs. potentially large skill injections). The tradeoff: internalized knowledge requires retraining to update, whereas external stores (SkillBank, learnings.md, CORAL Skills) can be extended at any time.

SKILL-0's weight internalization is conceptually adjacent to [sources/agentflow](../sources/agentflow.md)'s Flow-GRPO — both persist knowledge via weight updates — but SKILL-0 targets *skill knowledge* while AgentFlow targets *task policy*.

### Executable Skills (Parameterized Programs + NL Guidance) — [sources/webxskill](../sources/webxskill.md)

WebXSkill represents each skill as a **dual artifact**: a parameterized executable program *and* step-level natural-language guidance. The same skill can be deployed in two modes — grounded (auto-execute the program) or guided (feed NL instructions to the agent step-by-step).

Indexed in a **URL-based graph** rather than flat or hierarchical: retrieval is context-aware based on which part of the web the agent is currently navigating.

This is the first system in the wiki to make the NL/code duality an explicit design principle. Other skill systems either commit to one side (SKILL-0 internalizes as weights; Meta-Harness stores code) or pair them loosely (SKILL-RL, CORAL). WebXSkill's dual representation means the skill artifact is both executable and instructional without duplication.

### Per-Capability LoRA Adapters — [sources/trace](../sources/trace.md)

TRACE is the only system in the wiki that accumulates knowledge as a **modular set of weight deltas**: one LoRA adapter per identified capability gap. A router classifier selects which adapter to apply per task at inference.

Compared to other weight-based accumulation:
- [sources/agentflow](../sources/agentflow.md) updates one monolithic planner's weights in-the-flow
- [sources/skill-rl-skill0](../sources/skill-rl-skill0.md) SKILL-0 internalizes skills into the base model's weights via curriculum
- TRACE splits weight updates into **discrete, capability-isolated modules** with explicit routing

This gives TRACE an unusual property: new capabilities can be *added* without retraining existing adapters — orthogonal to monolithic weight updates which risk catastrophic forgetting.

### Skill Document as Trainable State — [sources/skillopt](../sources/skillopt.md)

SkillOpt makes the accumulation artifact explicit: a single markdown file (`best_skill.md`) *is* the trainable state of the system. The frozen model is unchanged; all accumulated knowledge lives in the document.

What distinguishes SkillOpt's accumulation from learnings.md ([sources/auto-harness](../sources/auto-harness.md)) or system-prompt search ([sources/honedhaiku](../sources/honedhaiku.md)):

- **Structured edit operators**: add / delete / replace within an **edit budget** (treated explicitly as a "textual learning rate"). The accumulation grows gradually, not via free-form rewrites
- **Rejected-edit buffer**: candidates that fail the validation gate are *retained* as negative examples for the optimizer — a second persistent store dedicated to failed mutations. Most other systems discard rejected proposals; SkillOpt mines them as hard negatives
- **Optimizer-side meta-skill**: the optimizer agent maintains its *own* slowly-updating skill document about how to write better edits — a small meta-evolution layer on top of the primary accumulation

The transfer evidence is the strongest in the wiki: `best_skill.md` produces +15.2% cross-model, +31.8% cross-harness, +10.4% self-optimizer gains without re-optimization. This argues that **structured prose skill documents are genuinely model- and harness-agnostic** as accumulated knowledge — a property weight-based accumulation ([sources/skill-rl-skill0](../sources/skill-rl-skill0.md) SKILL-0, [sources/trace](../sources/trace.md)) cannot match.

### Skill Instructions over an RLM Runtime — [sources/rlm-gepa](../sources/rlm-gepa.md)

[sources/rlm-gepa](../sources/rlm-gepa.md) also treats *skill instructions* as the accumulation artifact, but lays them on top of a fixed Recursive Language Model runtime (DSPy signatures + tools held constant). Transfer across use cases is the explicit design goal, with `AgentSpec` declaring the boundary within which transfer should hold.

The contrast with [sources/skillopt](../sources/skillopt.md) is instructive — they converge on the same artifact (a structured prose skill document) from different starting points:

- SkillOpt accumulates against a *frozen LLM*; bounded edits via "textual learning rate"
- RLM-GEPA accumulates against a *fixed RLM/DSPy structure*; surgical edits proposed by GEPA, scoped by AgentSpec

Both treat the prose layer as the unit of accumulation and the structured substrate underneath (model weights, or DSPy signatures and tools) as immutable. This convergence — independent groups arriving at "structured prose skill instructions are the natural granularity for accumulated knowledge" — is one of the strongest cross-source patterns in the wiki.

### Protocol-Native Lineage — [sources/autogenesis](../sources/autogenesis.md)

[sources/autogenesis](../sources/autogenesis.md) proposes lineage as a **protocol primitive** rather than an optimizer feature. Every modification to any resource (prompt, tool, memory, agent, environment) is versioned, attributed, and tagged with decision rationale. The accumulated store is the complete history of the agent's self-modifications, queryable and rollback-able at the protocol level.

This is closest to [sources/coral](../sources/coral.md)'s Attempts store (per-commit lineage) but generalizes across all agent internals, not just evaluated code commits. Where CORAL's lineage exists to enable multi-agent coordination via shared git, AGP's lineage exists to make self-modification *inspectable and reversible*.

## Shared Memory in Multi-Agent Systems

When multiple agents run in parallel, knowledge accumulation must address concurrent access. [sources/coral](../sources/coral.md) solves this with symlinks from isolated git worktrees to a centralized public directory. No locking needed — agents write asynchronously; reads are always consistent.

The result: emergent coordination behaviors arise from shared memory access alone. Agents copy each other's successful techniques (copycatting), synthesize patterns across agents' notes (cross-referencing), and eventually form consensus on what's been exhausted (agent consensus). See [concepts/self-improvement-loop](self-improvement-loop.md).

## The Consolidation Path: Files → Harness → Weights

[sources/self-evolving](../sources/self-evolving.md) (Xinming Tu) supplies a unifying axis over every store catalogued above: accumulated knowledge lives in one of three **substrates**, and discoveries migrate *upward* through them as they prove durable — a **consolidation path**.

| Substrate | Stores on this page | Durability / cost to update |
|-----------|---------------------|-----------------------------|
| **External files** | `learnings.md`, Cognition Base, CORAL Notes/Skills, `best_skill.md`, RLM-GEPA skill instructions, WebXSkill artifacts | Cheapest to extend; least durable/general |
| **Agent harness** | tool wiring, prompts, skill *structure* (the layer SkillOpt/RLM-GEPA edit) | Medium |
| **Model weights** | SKILL-0 internalization, TRACE LoRA adapters, AgentFlow planner | Most durable/general; costliest (retraining) |

The path explains several cross-source patterns the wiki had noted only pairwise:

- **SKILL-RL → SKILL-0** is a *file → weights* consolidation: skills accumulate externally in a SkillBank, then a curriculum internalizes them into weights once stable. Tu's framing names exactly this move.
- The **"structured prose skill document" convergence** ([SkillOpt](../sources/skillopt.md) + [RLM-GEPA](../sources/rlm-gepa.md)) is the field settling on the *file/harness boundary* as the sweet spot: durable enough to transfer (SkillOpt's +15.2% cross-model), cheap enough to edit every round.
- The tradeoff this page already draws (external stores extend anytime; weight internalization needs retraining) *is* Tu's durability-vs-cost gradient.

The consolidation lens also reframes the **Forgetting** open question below: pruning is deciding *when a discovery should stop consolidating upward* (or be demoted), not just deleting stale entries.

## Knowledge Accumulation vs. Feedback Signals

These are related but distinct:

| | Feedback signals | Knowledge accumulation |
|--|-----------------|----------------------|
| **Scope** | Current iteration | Across iterations |
| **Purpose** | Inform next proposal | Build a persistent knowledge base |
| **Form** | Scalar or rich trace | Structured documents, embeddings |
| **Author** | Evaluation infrastructure | The agent itself |

See [concepts/feedback-signals](feedback-signals.md) for the per-iteration side.

## Open Questions

- **Scale:** At what point does the knowledge base become too large to retrieve from efficiently? [sources/asi-evolve](../sources/asi-evolve.md) uses embedding-based semantic search; [sources/coral](../sources/coral.md) relies on agent judgment about what to read. Neither has been tested at thousands of hours of accumulated runs.
- **Forgetting:** Should outdated knowledge (from early in the run, before key insights) be pruned? None of the current systems address this.
- **Cross-domain transfer:** Can a knowledge base trained on one task transfer to another? [sources/asi-evolve](../sources/asi-evolve.md) seeds the Cognition Base from human literature; genuinely cross-task accumulation has not been demonstrated.

## Connections

- [concepts/self-improvement-loop](self-improvement-loop.md) — knowledge accumulation makes the loop compound rather than restart
- [concepts/feedback-signals](feedback-signals.md) — rich feedback is often the raw material that knowledge accumulation structures and stores
- [concepts/regression-gating](regression-gating.md) — the knowledge base can store which changes caused regressions, preventing re-testing failed approaches
- [sources/self-evolving](../sources/self-evolving.md) — the files → harness → weights consolidation path that organizes every store on this page

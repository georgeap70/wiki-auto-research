---
title: Context Engineering
type: concept
tags: [context-engineering, playbook, delta-updates, context-collapse, mechanism-vs-content, no-weight-updates, execution-state, context-poisoning, token-efficiency]
sources: [ace, mce, skillOpt, optimize-anything, halo, skill.state, wiki.skill]
last_updated: 2026-09-08
---

# Context Engineering

**Context engineering (CE)** is self-improvement by *evolving the model's context* — the structured, accumulated text a frozen model reads at inference — rather than its prompt wording, its weights, or its surrounding code. It is a distinct slice of [harness optimization](harness-optimization.md): the target is specifically the **persistent, structured context artifact** (a playbook, a cheatsheet, a set of bullets), and the discipline is about *how that artifact grows and is curated over time* without eroding.

The term is used explicitly in [Lilian Weng's survey](../sources/weng-harness-blog.md), which places CE on its optimization ladder (instruction → **structured context** → workflow → harness code → optimizer code) and names two exemplars the wiki now covers: [ACE](../sources/ace.md) and [MCE](../sources/mce.md).

## Content vs. Mechanism: Two Levels

The two CE systems in the wiki sit at different rungs of the same ladder:

| System | What it evolves | How |
|--------|-----------------|-----|
| [ACE](../sources/ace.md) (Agentic Context Engineering) | **Content** — an itemized playbook of bullet strategies | Fixed Generator → Reflector → Curator pipeline with deterministic delta merges |
| [MCE](../sources/mce.md) (Meta Context Engineering) | **Mechanism** — the CE *skill* (operators + code that manage context) | Bi-level (1+1)-ES; an LLM crossover operator evolves the skill itself |
| [SKILL.state](../sources/skill-state.md) | **Nothing — the substrate is replaced** | Hand-designed runtime: append-only history is swapped for a validated, mutable execution state; no search at all |

MCE explicitly frames ACE's fixed pipeline as **one point** in the space of possible context-management skills, and searches over the pipelines. This is the same content→mechanism jump that [EvoX](../sources/evox.md) makes for search strategies and [Hyperagents](../sources/hyperagents.md) makes for self-modification procedures — meta-evolution applied to context management.

[SKILL.state](../sources/skill-state.md) is a third position that dissolves the question rather than climbing the ladder: **if you fix the substrate correctly once, there is much less context to manage.** It is worth holding alongside ACE and MCE precisely because it is not an optimizer — it is the strongest evidence in the wiki that a *structural* choice about context can outperform searching over context-management policies.

## Replacing the Substrate: Bounded Execution State

[SKILL.state](../sources/skill-state.md) keeps only `(P, Σ_t, O_t)` in the prompt at every step — immutable skill spec, structured execution state, latest observation — and **discards the intermediate reasoning trace** the moment it has produced a validated state update `Σ_{t+1} = Σ_t ⊕ ΔΣ_t`. Prompt footprint is O(1) and cumulative tokens O(T), against O(T²) for conversational runtimes. Schema ownership and validation live in the deterministic runtime, so a malformed model patch cannot corrupt persistent state (invalid patches trigger rollback-retry).

Three of its results speak directly to this page's concerns:

**Context poisoning, quantified.** When the world changes outside the agent's action loop, history-based runtimes **hallucinate for 5–8 consecutive turns** — obsolete facts in the prompt history overpower contradictory new observations. SKILL.state needs **zero** recovery steps. Under injected distractor telemetry (5/20/50 events per turn) ReAct degrades 0.68 → 0.53 while SKILL.state holds ≥0.97, because distractors are filtered during patch generation and *never enter a later prompt*. This is a companion failure mode to context collapse: collapse is losing what you need, poisoning is retaining what you don't.

**Structure beats brevity — measurably.** Budget-matched to ~1,800 tokens (Warehouse, T=100):

| Configuration | Score |
|---------------|-------|
| Full ReAct (unbounded, 1.25M tokens) | 0.84 |
| Truncated sliding window | 0.18 |
| Summary-capped | 0.52 |
| ReAct + LLMLingua (perplexity compression) | 0.22 |
| **SKILL.state** | **0.94** |

The gain is not from shorter prompts. Statistical compression removes seemingly-redundant slot identifiers that are semantically vital; truncation evicts early allocations that matter later. This is the quantitative version of this page's central claim — *small, structured, additive edits accumulate; free-form rewriting or lossy compression erodes* — with statistical compressors added to the list of things that erode.

**Where it fails is the boundary of the whole idea.** The retained state must be a **sufficient statistic**. That fails when no schema is known in advance, when an earlier observation's relevance goes unrecognized at the time it is seen (discarded context cannot be revisited, unlike a transcript), or when the **history itself is the objective** — auditing, provenance, explaining past actions.

## Two Time Horizons, Two Opposite Disciplines

Put next to [WikiSkill](../sources/wikiskill.md), the wiki's newest CE-adjacent sources stake out opposite ends of one axis, and the apparent contradiction resolves cleanly:

| | Within a single execution | Across iterations / runs |
|--|---------------------------|--------------------------|
| Correct discipline | **Discard aggressively** into validated state ([SKILL.state](../sources/skill-state.md)) | **Never discard**; compound in a persistent store ([WikiSkill](../sources/wikiskill.md), [ACE](../sources/ace.md)) |
| Failure if you get it wrong | Context poisoning, quadratic cost, accuracy decay with horizon | Rediscovering the same failures every iteration (+15.0 points lost, per WikiSkill's ablation) |

They are complementary, and in fact compose: SKILL.state's immutable spec `P` is exactly the artifact a skill-evolution loop produces, and WikiSkill's rollouts would be cheaper and more reliable executed on a bounded-state runtime. The shared principle is that **what earns a place in the context must be explicitly justified** — by an item's helpful/harmful counters ([ACE](../sources/ace.md)), by a pattern page's accumulated evidence ([WikiSkill](../sources/wikiskill.md)), or by a schema field's necessity for future execution ([SKILL.state](../sources/skill-state.md)).

A practical caveat from [HarnessDev](../sources/harnessdev.md): asked to build harnesses from scratch, frontier models implement execution loops 18/18 times but **checkpoint state once in 18**, with zero checkpoint events across 26,679 trajectories. The context/state layer is simultaneously the highest-leverage and the least likely to be built without being asked for.

## The Central Failure Mode: Context Collapse

CE's defining problem is that naive approaches **destroy accumulated knowledge as they update**:

- **Context collapse** — when a system rewrites the *whole* context each step (e.g. Dynamic Cheatsheet), the rewrite silently erases hard-won detail. [ACE](../sources/ace.md) is designed against this: the Curator emits small **itemized delta updates** (append a bullet, or edit one in place), merged by **deterministic non-LLM logic**, so prior items survive.
- **Brevity bias** — prompt optimizers ([GEPA](../sources/optimize-anything.md)-style) tend to collapse toward short, generic prompts, dropping domain insight. Structured item lists resist this because each item is retained on its own merits (with helpful/harmful counters).

This is the same insight as [SkillOpt](../sources/skillopt.md)'s **bounded, structured edits** ("textual learning rate") and [evolve-the-harness](../sources/evolve-the-harness.md)'s preference for deterministic mechanisms: *small, structured, additive edits accumulate and transfer; free-form whole-artifact rewriting overfits or erodes.*

## What Is Optimized / Feedback / Gating

| Axis | Context engineering |
|------|---------------------|
| Optimization target | A persistent structured context artifact (no weight updates) |
| Feedback signal | Execution feedback; [ACE](../sources/ace.md) works with or without ground-truth labels |
| Loop | Generate → reflect → curate ([ACE](../sources/ace.md)); bi-level skill evolution over a base CE loop ([MCE](../sources/mce.md)) |
| Gating | Usually *structural* rather than threshold-based — deterministic delta merges + dedup preserve prior knowledge; MCE keeps the better of {incumbent, one offspring} on validation |

## Efficiency as a Headline Result

Both systems report large efficiency wins over prompt-optimization baselines, not just accuracy:
- [ACE](../sources/ace.md): 82.3% lower adaptation latency and 75.1% fewer rollouts than [GEPA](../sources/optimize-anything.md) (offline AppWorld); 91.5% lower latency and 83.6% lower token cost than Dynamic Cheatsheet (online FiNER).
- [MCE](../sources/mce.md): ~13.6× faster training and ~4.8× fewer rollouts than baselines on FiNER.

Item-level delta editing avoids re-deriving the whole context every step — the efficiency argument for structured accumulation over rewriting.

## Relation to Other Wiki Concepts

- **[Knowledge accumulation](knowledge-accumulation.md)** — CE *is* a form of knowledge accumulation; the playbook/skill is the accumulated artifact. The distinctive contribution is the *anti-collapse* discipline for how it grows.
- **[Harness optimization](harness-optimization.md)** — CE optimizes the context layer of the harness specifically.
- **[Feedback signals](feedback-signals.md)** — [ACE](../sources/ace.md)'s Reflector is a feedback-distillation component, and it can run label-free.
- **[Evolutionary optimization](evolutionary-optimization.md)** — [MCE](../sources/mce.md) uses an evolutionary meta-loop with an LLM crossover operator.

## Connections

- [sources/ace](../sources/ace.md) — content-evolving CE (Generator/Reflector/Curator, delta updates)
- [sources/mce](../sources/mce.md) — mechanism-evolving CE (bi-level, agentic crossover)
- [sources/skillopt](../sources/skillopt.md) — sibling "structured artifact, bounded edits" approach at the skill-document layer
- [sources/skill-state](../sources/skill-state.md) — substrate-replacing CE: validated bounded execution state instead of accumulating history
- [sources/wikiskill](../sources/wikiskill.md) — the across-iteration pole; patch-based pattern editing with the same anti-erosion discipline
- [sources/weng-harness-blog](../sources/weng-harness-blog.md) — names context engineering as a harness-optimization category
- [concepts/knowledge-accumulation](knowledge-accumulation.md), [concepts/harness-optimization](harness-optimization.md)

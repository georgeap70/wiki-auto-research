---
title: SkillOpt — Skill Document as Trainable State
type: source
tags: [skills, prompt-optimization, frozen-model, bounded-edits, optimizer-target, microsoft]
sources: [skillOpt]
last_updated: 2026-04-28
---

# SkillOpt

**Project page**: [microsoft.github.io/SkillOpt](https://microsoft.github.io/SkillOpt/)
**Paper**: arXiv:2605.23904
**Authors**: Microsoft Research (companion project: SkillLens)

## One-line summary

Treats a compact natural-language **skill document** as the trainable state of a frozen LLM, optimized via a reflection + bounded-edit loop with a validation gate. One file (`best_skill.md`) is the entire deliverable; works across 7 models × 6 benchmarks with best-or-tied-best on all 52 combinations.

## Core idea

In standard ML, weights are the trainable state. In SkillOpt, the trainable state is a markdown document — a structured set of procedural instructions that the frozen target model reads at runtime. Optimization changes the *document*, not the model.

This is a refinement of the harness-optimization paradigm: rather than evolving system prompts ([sources/honedhaiku](honedhaiku.md)) or full agent code ([sources/evoforge](evoforge.md), [sources/autoagent-kevinrgu](autoagent-kevinrgu.md)), SkillOpt evolves a **structured skill artifact** with explicit add/delete/replace operators and a "textual learning rate" governing edit magnitude.

## Loop architecture

```
1. Rollout       — frozen target model runs tasks with current skill
                   → scored trajectories (messages, tool calls, feedback)
2. Reflection    — optimizer model analyzes success and failure batches
                   independently → identifies reusable procedures
3. Bounded Edits — optimizer proposes add/delete/replace ops within
                   an edit budget (textual learning rate)
4. Validation    — candidate skill kept only if held-out perf improves
```

The clean **target / optimizer separation** echoes [sources/halo](halo.md) (RLM analyzer vs. coding agent) and [sources/autoreason](autoreason.md) (incumbent vs. judges) — three different specialized-component architectures for diagnosis-vs-editing decoupling.

## What's novel

### Edit budget as textual learning rate
The optimizer can only propose a bounded number of operations per round. This is conceptually identical to gradient-descent step size: small budget prevents overwriting functional rules; large budget allows fast escape from poor local minima. To my knowledge this is the first explicit framing of textual edit size as a learning-rate hyperparameter.

### Rejected-edit buffer (negative signal)
Edits that fail the validation gate are *stored* and fed back to the optimizer as negative examples. Most other systems in the wiki discard rejected proposals; SkillOpt mines them. This is structurally similar to hard-negative mining in contrastive learning.

### Optimizer-side meta-skill
The optimizer model itself maintains a *meta-skill* — instructions about how to write better skill edits — that updates more slowly than the target skill. This is a small meta-evolution layer (cf. [sources/evox](evox.md)) but applied to the optimizer rather than the search strategy.

### Separate success/failure analysis
Reflection analyzes successful batches and failed batches *independently* before merging proposals. This avoids the bias toward fixing failures at the expense of preserving wins — a subtler version of the prompt-bias problem identified by [sources/autoreason](autoreason.md).

## Key results

- **52/52 best-or-tied-best** across 7 target models × 6 benchmarks
- Models tested: GPT-5.5, GPT-5.4, GPT-5.4-mini, GPT-5.4-nano, GPT-5.2, and two Qwen variants
- Benchmarks: SearchQA, Sheet, Office, DocVQA, LiveMath, ALFWorld
- Average gains: **9–25%** across settings
- ALFWorld: **70.9% → 85.8%** (+14.9pp)

### Transfer

The exported `best_skill.md` shows substantial transfer without re-optimization:

| Transfer type | Gain |
|---------------|------|
| Cross-model (skill from model A used on model B) | +15.2% |
| Cross-harness (skill from harness A used in harness B) | +31.8% |
| Self-optimizer (skill replaces optimizer's meta-skill) | +10.4% |

This is the strongest transfer evidence in the wiki: optimized skill docs are not model- or harness-specific.

## Connections

- [sources/skill-rl-skill0](skill-rl-skill0.md) — both call their unit a "skill", but the storage differs: SKILL-RL maintains an external SkillBank consulted during RL; SKILL-0 internalizes skills into weights via curriculum; SkillOpt evolves a single skill *document* with no weight changes. SkillOpt is the lightest-weight of the three
- [sources/webxskill](webxskill.md) — WebXSkill's skills are *executable programs* with NL guidance; SkillOpt's skills are pure prose. WebXSkill compiles to code; SkillOpt stays at the prompt layer
- [sources/honedhaiku](honedhaiku.md) — both evolve text without touching weights; HonedHaiku optimizes a system prompt via GEPA mutations, SkillOpt evolves a skill doc via bounded reflection-driven edits. SkillOpt's gains span weaker models too (the Goldilocks band may be wider when edits are structured)
- [sources/halo](halo.md), [sources/autoreason](autoreason.md) — share the *optimizer separated from target* architecture; SkillOpt adds rejected-edit memory on the optimizer side
- [sources/optimize-anything](optimize-anything.md) — GEPA also operates on text via LLM mutations; SkillOpt restricts the edit space (bounded add/delete/replace) rather than allowing free-form mutation
- [concepts/harness-optimization](../concepts/harness-optimization.md) — SkillOpt sits at the structured-prompt sub-layer of harness optimization
- [concepts/knowledge-accumulation](../concepts/knowledge-accumulation.md) — `best_skill.md` is the persistent accumulated knowledge; rejected-edit buffer is a secondary persistent store
- [concepts/regression-gating](../concepts/regression-gating.md) — validation-on-held-out is the gate; same primitive as [sources/meta-harness](meta-harness.md)

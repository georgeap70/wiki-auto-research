---
title: "ACE: Agentic Context Engineering (Evolving Contexts for Self-Improving LMs)"
type: source
tags: [context-engineering, playbook, delta-updates, context-collapse, generator-reflector-curator, no-weight-updates]
sources: [ace]
url: https://arxiv.org/abs/2510.04618
code: https://github.com/ace-agent/ace
authors: Qizheng Zhang, Changran Hu, Shubhangi Upasani, et al.; James Zou, Kunle Olukotun
org: Stanford + SambaNova Systems + UC Berkeley
arxiv: 2510.04618
venue: ICLR 2026
last_updated: 2026-07-10
---

# ACE: Agentic Context Engineering (arXiv 2510.04618)

## Summary

ACE treats the model's **context as an evolving, itemized "playbook"** — a structured list of bullet strategies, each with an id and helpful/harmful counters — rather than one monolithic prompt that gets rewritten. It is the wiki's cleanest example of [context engineering](../concepts/context-engineering.md): self-improvement by *accumulating and curating structured context*, with **no weight updates**. Its key contribution is a mechanism to avoid **context collapse** — the failure where whole-context rewriting erodes accumulated knowledge — that plagues prompt-rewriting optimizers.

## Core Method — Generator / Reflector / Curator

- **Generator** produces reasoning trajectories for tasks and surfaces which context items were useful or missing.
- **Reflector** analyzes successes and failures, distilling concrete lessons.
- **Curator** synthesizes those lessons into small, itemized **delta updates** and merges them into the playbook via **deterministic (non-LLM) logic**.

**Grow-and-refine**: deltas are appended or edit a single bullet in place (never a full rewrite); merges are deterministic, enabling parallel learning across samples; periodic embedding-based de-duplication/pruning keeps the playbook from bloating. This design explicitly defeats two failure modes:
- **Context collapse** — monolithic rewriting silently erasing detail (the pathology of whole-context rewriters like Dynamic Cheatsheet).
- **Brevity bias** — optimizers collapsing toward short/generic prompts and dropping domain insight.

## What Is Optimized / Feedback / Loop / Gating

| Axis | ACE |
|------|-----|
| Optimization target | The context/playbook only (no weights) |
| Feedback signal | Execution feedback; works **with or without ground-truth labels** (enables online self-improvement) |
| Loop structure | Generator → Reflector → Curator; offline (batch, multi-epoch) *and* online (test-time) |
| Gating/safety | No threshold gate; anti-erosion is *structural* — deterministic delta merges + grow-and-refine dedup preserve prior bullets; per-item helpful/harmful counters |

## Results

- **Agents: +10.6%** average over strong baselines (AppWorld up to +17.1%). **Finance/domain: +8.6%** (FiNER, Formula).
- **AppWorld leaderboard**: matches the top-ranked production agent on overall average and *surpasses* it on the harder test-challenge split — using a **smaller open-source model**.
- **Efficiency**: offline AppWorld — 82.3% lower adaptation latency and 75.1% fewer rollouts vs [GEPA](optimize-anything.md). Online FiNER — 91.5% lower adaptation latency and 83.6% lower token cost vs Dynamic Cheatsheet (which rewrites the whole context each step; ACE updates item-by-item).

## Why It Matters

- **Complements the wiki's prompt-optimizers.** [HonedHaiku](honedhaiku.md) and [GEPA](optimize-anything.md) mutate prompt *prose*; ACE instead maintains a *structured, additively-curated* context. The efficiency wins vs GEPA/Dynamic Cheatsheet argue that item-level delta editing beats whole-context rewriting — the same "bounded, structured edits transfer/scale" theme as [SkillOpt](skillopt.md)'s textual learning rate.
- **Context collapse** is a named, general failure mode worth tracking wherever a system rewrites accumulated text (cf. [ASI-Evolve](asi-evolve.md)'s Analyzer, [SkillOpt](skillopt.md)).

## Connections

- [concepts/context-engineering](../concepts/context-engineering.md) — ACE is the content-evolving exemplar of this family
- [concepts/knowledge-accumulation](../concepts/knowledge-accumulation.md) — the playbook is the accumulated artifact; delta updates + dedup are the accumulation discipline
- [concepts/harness-optimization](../concepts/harness-optimization.md) — optimizes the context layer of the harness, no weights
- [concepts/feedback-signals](../concepts/feedback-signals.md) — Reflector distills execution feedback; works label-free
- [sources/mce](mce.md) — evolves the *mechanism* that manages context, treating ACE's fixed pipeline as one point in a larger space
- [sources/skillopt](skillopt.md) — sibling "structured artifact, bounded edits" approach at the skill-document layer
- [sources/weng-harness-blog](weng-harness-blog.md) — cites ACE as the context-engineering exemplar

---
title: "Hyperagents (DGM-H): Metacognitive Self-Improving Agents"
type: source
tags: [self-modifying-code, meta-agent, open-ended-evolution, metacognition, any-computable-task, meta]
sources: [hyperagents]
url: https://arxiv.org/abs/2603.19461
code: https://github.com/facebookresearch/HyperAgents
authors: Jenny Zhang, Bingchen Zhao, Wannan Yang, Jakob Foerster, Jeff Clune, Minqi Jiang, Sam Devlin, Tatiana Shavrina
org: Meta + University of British Columbia
arxiv: 2603.19461
last_updated: 2026-07-10
---

# Hyperagents / DGM-H (arXiv 2603.19461)

## Summary

Hyperagents (a.k.a. **DGM-H**, Darwin Gödel Machine–Hyperagent) is the direct follow-up to the [Darwin Gödel Machine](dgm.md) by an overlapping author group (Jenny Zhang, Jeff Clune). Its key move: **merge the task agent and the meta-agent into one self-modifiable codebase**, so the meta-agent can edit *itself* — including its own modification procedure. This "metacognitive" self-improvement is meant to generalize self-improvement to **any computable task**, not just coding.

## Core Method

- **Single editable program** fuses the *task agent* (solves the target task) with the *meta-agent* (improves the task agent). Because the meta-agent's edit procedure is itself editable, the system can improve *how it improves* — the structural step beyond DGM.
- **Loop**: classic DGM-style open-ended archive — meta-agent reads the code, analyzes past performance, generates a modified version, evaluates it, and archives it if better; new parents are drawn from the archive.
- **Parent selection**: [Weng](weng-harness-blog.md) ascribes the DGM family rule (probability ∝ performance and ∝ 1/offspring) to this cluster; the exact formula was not confirmed from the abstract, so treat it as inherited-from-[DGM](dgm.md).

## Relation to the Darwin Gödel Machine

- **Extends** DGM's generate–evaluate–archive framework.
- **Removes DGM's key assumption** that task-solving skill and self-modification skill are aligned (i.e., that getting better at coding = getting better at self-improvement). By adding a *separately editable* meta-agent whose modification procedure is editable, DGM-H aims for self-improvement across arbitrary computable tasks.

## What Is Optimized / Feedback / Loop / Gating

| Axis | Hyperagents (DGM-H) |
|------|---------------------|
| Optimization target | Both the task agent *and* the meta-agent's own improvement strategy |
| Feedback signal | Measured per-domain task-performance improvements |
| Loop structure | Open-ended archive evolution (DGM-style), with a self-editable meta-agent |
| Gating/safety | Empirical per-candidate evaluation; better variants archived (regression-style check) |

## Results (qualitative; exact numbers not in the abstract)

- Domains: **coding, paper review, robotics reward design, and Olympiad/math grading** — deliberately spanning beyond coding to test the any-computable-task claim.
- Reported: DGM-H "improves performance over time and outperforms baselines"; improvements "transfer across domains and accumulate across runs." Agents autonomously evolved persistent memory, performance tracking, multi-stage verification pipelines, decision protocols, and retry logic.
- Quantitative figures should be confirmed from the full PDF.

## Disambiguation

**Not** the older, unrelated **HyperAgent** (Phan et al., FSoft-AI4Code, arXiv 2409.16299, Sept 2024) — a fixed multi-agent SWE system (Planner/Navigator/Editor/Executor). Different authors, org, year, and purpose; the name collision is coincidental.

## Why It Matters

- Pushes one rung further up [Weng's ladder](weng-harness-blog.md): from optimizing the harness to optimizing the *optimizer's own procedure* — the wiki's [meta-level self-improvement open question](../overview.md) made concrete.
- The multi-domain evaluation (robotics, grading, review) is a direct test of whether self-modifying-agent gains are coding-specific or general.

## Uncertainty Notes

Affiliation (Meta + UBC) inferred from secondary coverage and the `facebookresearch/` repo namespace; the arXiv abstract did not print affiliations. Parent-selection formula and quantitative numbers unverified from the abstract.

## Connections

- [sources/dgm](dgm.md) — the parent system; DGM-H adds a self-editable meta-agent and drops the task-skill = self-improvement-skill assumption
- [sources/stop](stop.md) — the original "improve the improver"; DGM-H is its most literal modern realization (edit the modification procedure itself)
- [sources/adas](adas.md) — meta-agent-over-archive lineage
- [concepts/self-improvement-loop](../concepts/self-improvement-loop.md) — metacognitive/meta-agent self-modification
- [concepts/evolutionary-optimization](../concepts/evolutionary-optimization.md) — open-ended archive evolution
- [concepts/harness-optimization](../concepts/harness-optimization.md) — evolves its own harness *and* the procedure that evolves it
- [sources/weng-harness-blog](weng-harness-blog.md) — cites Hyperagents as a DGM follow-up introducing a controlling meta-agent

---
title: "ADAS: Automated Design of Agentic Systems (Meta Agent Search)"
type: source
tags: [agent-design, meta-agent, code-as-search-space, archive, transfer, foundational]
sources: [adas]
url: https://arxiv.org/abs/2408.08435
code: https://github.com/ShengranHu/ADAS
authors: Shengran Hu, Cong Lu, Jeff Clune
org: University of British Columbia / Vector Institute (Clune lab)
arxiv: 2408.08435
venue: ICLR 2025
last_updated: 2026-07-10
---

# ADAS: Automated Design of Agentic Systems (arXiv 2408.08435)

## Summary

ADAS proposes **Meta Agent Search**: a meta-agent that *programs new agents in code*, tests them, and accumulates them in an ever-growing **archive** that conditions the next round. Its foundational move is to make the search space **all of code** — because code is Turing-complete, the space in principle covers any prompt, tool use, control flow, or combination, rather than tuning within a fixed framework. This is the direct ancestor of the wiki's harness-population and self-designing-agent systems ([EvoForge](evoforge.md), [AutoAgent (KevinRGU)](autoagent-kevinrgu.md), [DGM](dgm.md)).

## Core Method

1. A **meta-agent** writes a new agent as executable code (novel building blocks + composition), conditioned on the archive of previously discovered agents.
2. The new agent is evaluated on target tasks; performance (with 95% bootstrap CIs) is fed back.
3. The agent is added to the archive; archive-conditioning steers the next proposal toward *interesting new* agents (novelty).
4. Generated agents themselves often include multi-step **self-refinement** loops (~3 iterations).

## What Is Optimized / Feedback / Loop

| Axis | ADAS |
|------|------|
| Optimization target | Whole agentic systems expressed as code (prompts, tools, control flow) |
| Feedback signal | Held-out task accuracy (bootstrap CIs); tolerates rich code-execution feedback |
| Loop structure | Single meta-agent + growing archive (open-ended / evolutionary flavor) |
| Gating/safety | Held-out test evaluation; **executes untrusted model-generated code** — sandboxing is the user's responsibility (explicit repo warning) |
| Human involvement | Define search domain + eval; then autonomous |

## Results

- Four domains, each with its own search: **ARC** (abstraction/reasoning showcase), **DROP** (reading comprehension), **MGSM** (multilingual math), **MMLU** (multitask knowledge).
- Discovered agents **outperform state-of-the-art hand-designed baselines in every domain**.
- **Transfer**: discovered agents keep their edge when moved *across domains* and *across models* (e.g., agents found on math transfer to non-math domains) — an early, strong result that good agent *designs* are not model- or task-specific.

(Exact per-benchmark percentages live in the paper's result tables.)

## Why It Matters

- **Code as the universal search space** is the idea the rest of the wiki's harness/workflow optimizers inherit. [Meta-Harness](meta-harness.md), [EvoForge](evoforge.md), and [DGM](dgm.md) all narrow or specialize this move.
- The **archive-conditioned meta-agent** is the template for open-ended agent evolution later formalized by [DGM](dgm.md)/[Hyperagents](hyperagents.md).
- Cross-domain/cross-model transfer of *designs* is an early counterpart to the wiki's later transfer findings ([SkillOpt](skillopt.md) prose; [evolve-the-harness](evolve-the-harness.md) code).

## Connections

- [concepts/self-improvement-loop](../concepts/self-improvement-loop.md) — meta-agent + archive; agents are proposed as code
- [concepts/harness-optimization](../concepts/harness-optimization.md) — designs whole agents/harnesses in code
- [concepts/evolutionary-optimization](../concepts/evolutionary-optimization.md) — archive-based open-ended search with novelty pressure
- [sources/aflow](aflow.md) — a more structured successor (search over *workflows* via MCTS rather than free-form code)
- [sources/dgm](dgm.md), [sources/hyperagents](hyperagents.md) — extend the meta-agent-over-archive idea to self-modifying agents
- [sources/weng-harness-blog](weng-harness-blog.md) — cites ADAS under "workflow design" harness optimization

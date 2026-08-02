---
title: "Optimize Anything Omni: Engine-Pluggable, Pipeline-Composable Optimization (GEPA)"
type: source
tags: [universal-optimization, GEPA, meta-optimizer, portfolio, engine-pluggable, composition, autoresearch, meta-harness, terrarium]
sources: [optimize-anything-omni]
url: https://gepa-ai.github.io/gepa/blog/2026/07/22/optimize-anything-omni/
code: https://github.com/gepa-ai/gepa
org: GEPA team (UC Berkeley / Stanford lineage)
published: 2026-07-22
last_updated: 2026-08-01
---

# Optimize Anything Omni (GEPA)

**Blog**: [Optimize Anything Omni](https://gepa-ai.github.io/gepa/blog/2026/07/22/optimize-anything-omni/) (2026-07-22)
**Code**: [github.com/gepa-ai/gepa](https://github.com/gepa-ai/gepa)

Direct successor to [sources/optimize-anything](optimize-anything.md). The original post argued *"anything text-representable with a measurable score can be optimized"* and shipped GEPA as the one optimizer behind that API. This update **decouples the API from the optimizer**, making the choice of optimizer a pluggable, composable dimension — and then meta-optimizes over that choice.

## One-line summary

`optimize_anything` becomes **engine-pluggable** (`engine="gepa" | "autoresearch" | "meta_harness"`) and **pipeline-composable** (chain / race / vote engines), and introduces **omni** — a meta-optimizer that runs a *portfolio* of engines in parallel then continues the winner with a fresh optimizer. Empirical punchline on Frontier-CS: **no single optimizer dominates, but every omni composition beats every standalone optimizer.**

## Why this matters for the wiki

The three pluggable engines are **three systems this wiki already tracks separately**, unified under one contract:

| Engine | Who proposes the next candidate | Who owns the loop | Wiki page |
|--------|--------------------------------|-------------------|-----------|
| **GEPA** (LLM-based) | A single reflective LLM call mutates a parent drawn from a Pareto frontier | External framework | [sources/optimize-anything](optimize-anything.md) |
| **AutoResearch** (autonomous agent) | A long-horizon Claude Code session; the agent owns selection, proposal, *and* evaluation | The agent itself | [sources/autoresearch-vs-hpo](autoresearch-vs-hpo.md) |
| **Meta-Harness** (agent-based) | A coding-agent proposer mutates candidates | Framework keeps outer-loop control + parent selection | [sources/meta-harness](meta-harness.md) |

This is the first source in the wiki to treat *"which self-improvement architecture do I use"* as itself an optimizable/composable choice. It operates at the **"optimizer-code" top of [Weng's ladder](weng-harness-blog.md)** (instruction → context → workflow → harness code → optimizer code) — but rather than *evolving* optimizer code, it *portfolios and schedules* whole optimizer families.

## The common contract

Every engine fits the same `(candidate, score, loop)` shape and differs only in *who proposes* and *who orchestrates*:

- **Evaluator polymorphism**: `(candidate) -> (score, info)` for single-task optimization, or `(candidate, example) -> (score, info)` for dataset tasks.
- **`info` dict = Actionable Side Information (ASI)** — the same [rich-feedback](../concepts/feedback-signals.md) channel from the original GEPA post; every engine's proposer reads it to decide what to mutate.
- **Custom engines** register in one line: `register_engine("my_engine", MyEngine)`, where the engine implements `__init__(config)` and `run(task, server) -> Result`. Registered engines immediately work with every composition helper and with Terrarium.

## The omni meta-optimizer

Omni runs in **two phases** on a fixed budget (the post uses $20/problem):

1. **Explore** — run all three engines in **parallel**, each spending ~$5, then keep the single highest-scoring candidate.
2. **Continue** — seed a **fresh** optimizer instance with the Phase-1 winner and spend the remaining ~$5 to push past the plateau.

The motivating observation: *"Given enough budget, every optimizer makes most of its gains early and then plateaus; the rest of the budget makes little difference."* A fresh optimizer handed a stalled candidate frequently makes further progress — so **portfolio-then-restart** beats spending the whole budget in one engine.

This is a budgeted, one-level-up version of the [sources/evox](evox.md) insight ("no single search strategy dominates across all tasks and phases"): no single *optimizer* dominates across tasks, so race them and continue the leader.

## Composition primitives

Beyond omni, the API exposes reusable combinators:

| Primitive | Behavior |
|-----------|----------|
| `optimize_sequential` | Chain engines; each continues the previous winner (monotonic) |
| `optimize_best_of` | Run engines in parallel, keep the highest score |
| `optimize_parallel` | Run engines in parallel, return all results |
| `optimize_vote` | Parallel, then cross-engine comparison via vote |
| `optimize_adaptive_sequential` | Auto-switch engines on **plateau detection** |

`optimize_adaptive_sequential` is the closest analog to [sources/evox](evox.md)'s stagnation-triggered strategy switching — but the switched unit is a whole engine, not a sampling strategy.

## Terrarium (controlled evaluation harness)

To compare engines fairly the team released **Terrarium**, which pins every optimizer to identical tasks, budgets, models (**Claude Sonnet 4.6, medium thinking**), and evaluation servers. This is the apparatus that makes "no single optimizer dominates" credible rather than anecdotal — a shared-substrate benchmark for *optimizers*, analogous to how [sources/meta-harness](meta-harness.md) fixed a benchmark to compare harness edits.

## Frontier-CS results

**Frontier-CS**: 10 competitive-programming problems, $20 budget each.

| Optimizer | Standalone | Omni variant | Δ |
|-----------|-----------:|-------------:|--:|
| GEPA | 43.8 | **61.8** | +41% |
| AutoResearch | 55.4 | **63.2** | +14% |
| Meta-Harness | 50.9 | **59.3** | +16% |

Two findings:

1. **No single optimizer dominates** — each engine wins roughly a third of problems, unpredictably.
2. **Every omni variant outscores every standalone optimizer.** Omni-AutoResearch is best overall (63.2); omni-GEPA (61.8) alone narrowly beats *standalone* AutoResearch (55.4) — wrapping the weakest standalone engine in omni made it competitive with the strongest standalone. The largest lift lands on GEPA (+41%), the engine most prone to early plateau — consistent with the "fresh optimizer breaks plateaus" thesis.

## What is optimized / what is not

**Optimized**: the target artifact (code, prompt, config — anything text-representable with a score), *and* — via omni — the portfolio/sequence of optimizers applied to it.

**Not optimized**: the base LLM weights (Sonnet 4.6 is a frozen operator throughout), and the omni schedule itself (the two-phase explore/continue split and the equal budget shares are fixed policy, not learned). Unlike the [self-modifying-code lineage](../concepts/evolutionary-optimization.md) ([STOP](stop.md) → [DGM](dgm.md) → [Hyperagents](hyperagents.md)), omni does **not** rewrite optimizer code — it selects and reseeds among fixed engines. It is a **meta-scheduling** layer over the optimizer choice, not a self-modifying optimizer.

## Positioning in this wiki

- Supersedes/extends [sources/optimize-anything](optimize-anything.md): GEPA is now *one engine among several* behind the same API.
- Unifies [sources/optimize-anything](optimize-anything.md) (GEPA), [sources/autoresearch-vs-hpo](autoresearch-vs-hpo.md) (AutoResearch), and [sources/meta-harness](meta-harness.md) (Meta-Harness) as interchangeable engines — a synthesis the wiki previously described only as three separate loop architectures.
- Peer to [sources/evox](evox.md) and [sources/mce](mce.md): all three say "no single X dominates, adapt/portfolio across X" — at the search-strategy level (EvoX), the context-management-mechanism level (MCE), and the whole-optimizer level (omni).
- Most directly relevant to [wiki/experiment.md](../experiment.md), whose committed design is a **single** GEPA loop — omni is the external data point arguing a portfolio-then-continue schedule may beat any single-engine run.

## Connections

- [sources/optimize-anything](optimize-anything.md) — the framework this generalizes; ASI, Pareto, GEPA-as-genetic-Pareto all carry over
- [sources/autoresearch-vs-hpo](autoresearch-vs-hpo.md) — the "AutoResearch" engine (Claude-Code-agent-owns-the-loop)
- [sources/meta-harness](meta-harness.md) — the "Meta-Harness" engine (framework-owned outer loop, agent proposer)
- [sources/evox](evox.md), [sources/mce](mce.md) — meta-evolution / "adapt across searchers," one level down
- [sources/aflow](aflow.md) — its MCTS soft-mixed selection *always retains the initial workflow*; omni's "keep a fresh optimizer in reserve" is a portfolio cousin of that hedge
- [sources/weng-harness-blog](weng-harness-blog.md) — omni sits at the optimizer-code rung of Weng's ladder
- [concepts/evolutionary-optimization](../concepts/evolutionary-optimization.md) — portfolio/meta-optimizer at the top of the hierarchy of evolution
- [concepts/feedback-signals](../concepts/feedback-signals.md) — the `info`/ASI channel is shared across all engines
- [wiki/experiment.md](../experiment.md) — "run one GEPA loop" vs. omni's "race engines, continue the winner"

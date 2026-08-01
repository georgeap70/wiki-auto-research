---
title: Squeeze-Evolve
type: source
tags: [evolution, test-time-scaling, multi-model, cost-aware, routing, verifier-free, inference, RSA, openevolve]
sources: [squeeze-evolve]
url: https://github.com/squeeze-evolve/squeeze-evolve
code: https://github.com/squeeze-evolve/squeeze-evolve
org: Squeeze-Evolve authors (COLM 2026)
published: 2026
last_updated: 2026-07-31
---

# Squeeze-Evolve

**Code / paper**: [github.com/squeeze-evolve/squeeze-evolve](https://github.com/squeeze-evolve/squeeze-evolve) — accepted to **COLM 2026**.

## One-line summary

A **verifier-free evolutionary framework with multi-model orchestration** for test-time scaling: it runs an evolutionary refinement loop over candidate *answers* and **routes each refinement step to a cost-appropriate model based on per-problem difficulty** — hard groups go to expensive models, easy groups to cheap ones — achieving "equivalent or better accuracy at a fraction of the cost."

## What it optimizes (and where it sits)

Unlike [sources/alphaevolve](alphaevolve.md) and [sources/shinkaevolve](shinkaevolve.md), which evolve **code/programs**, Squeeze-Evolve evolves the **task answer itself at inference time**. It is a *test-time scaling* method — the population is a set of candidate solutions to one problem, refined over loops. In [concepts/harness-optimization](../concepts/harness-optimization.md) terms nothing durable is optimized; the "improvement" is the per-query output, in the same time-axis family as [sources/autoreason](autoreason.md)'s per-query tournament — but population-based and cost-routed rather than tournament-gated.

## Core method: fitness-based routing

Each iteration runs five stages:

1. **Score** — compute a **zero-cost fitness proxy** for each candidate group: either *group confidence* from token log-probabilities (`prompt_logprobs: true`) or *answer diversity* (count of unique answers). No verifier, no ground truth, no reward model — hence "verifier-free."
2. **Select** — group candidates into subsets of size `K` via uniform or fitness-weighted sampling.
3. **Route** — split groups into `N` tiers using **per-problem adaptive percentile thresholds**; each tier maps to a model tier (expensive → hard groups, cheap → easy groups). For a 2-model setup, `confidence_percentiles: [50.0]` is the single split point.
4. **Recombine** — each model generates refined candidates in parallel; groups already at full consensus are shortcut with lightweight aggregation (majority vote) rather than a fresh LLM call.
5. **Update** — `replace` or `accumulate` candidates into the next loop.

**Loop 0** initializes by having the expensive model generate `population` initial candidates per problem; **loops 1..T** run the five-stage cycle (`loops` configurable).

The whole thing is a **single integrated population** shared across model tiers — not co-evolution of separate populations. The models are ordered cheapest → most expensive, and the percentiles define the routing boundaries between them.

## Feedback signal

The distinctive choice: fitness is a **zero-cost, verifier-free proxy for difficulty**, not a measured quality score.

- **Confidence** — token log-probs of the generation (high agreement ⇒ easy ⇒ route cheap).
- **Diversity** — unique-answer count (high spread ⇒ hard ⇒ route expensive).

This contrasts with every graded-fitness system in the wiki (`evaluate.py` scores in [sources/shinkaevolve](shinkaevolve.md), grader scores in [sources/coral](coral.md)). It is closest in spirit to [sources/autoreason](autoreason.md)'s *comparison-not-critique* signal and to consensus-based self-consistency — see [concepts/feedback-signals](../concepts/feedback-signals.md).

## Configuration / terminology

Key YAML knobs:

| Param | Meaning |
|-------|---------|
| `k` | group size |
| `population` | candidates per problem at loop 0 |
| `loops` | number of evolutionary iterations |
| `confidence_percentiles` | `N-1` thresholds for `N`-model routing |
| `fitness` | `confidence` \| `diversity` |
| `selection` | `uniform` \| `weighted` |
| `update` | `replace` \| `accumulate` |
| `lite_fraction` | fraction of easiest groups shortcut to lightweight aggregation |
| `recombination` | `aggregate` (task-aware) \| `synthesize` |

## Execution

- **CLI**: `squeeze-evolve-client --config benchmarks/aime25/configs/example.yaml --input data/aime25/test.parquet --n-problems 5`
- Ready-to-run benchmark scripts: `bash benchmarks/aime25/run.sh`
- **Deployments**: integrated into **NVIDIA Dynamo** as `dynamo.squeeze_evolve`; **Claude Code plugin** via `/plugin install squeeze-evolve@squeeze-evolve`.

## Evaluated tasks

**AIME 2025**, **HMMT 2025**, **GPQA-Diamond** (hard math + graduate science QA). The README references a scaling chart but does not print headline accuracy numbers; the claim is Pareto-style: same-or-better accuracy at a fraction of the cost.

## Relation to prior work

The README explicitly credits:

- **RSA (Recursive Self-Aggregation)** — the evolutionary aggregation loop and recombination prompts are inspired by RSA.
- **[OpenEvolve](alphaevolve.md)** — referenced as related evolutionary-coding infrastructure (the open AlphaEvolve reimplementation also named on [sources/alphaevolve](alphaevolve.md)).

It does **not** discuss AlphaEvolve, ShinkaEvolve, or GEPA directly, though it positions itself within the "evolutionary inference has strong test-time scaling" line.

## Squeeze-Evolve vs. ShinkaEvolve — two flavors of cost-aware multi-model

Both route across a cheap↔expensive model ensemble, but at **different layers of the loop** — a useful contrast for [wiki/experiment.md](../experiment.md)'s `[prompt, model]` question:

| Dimension | [ShinkaEvolve](shinkaevolve.md) | Squeeze-Evolve |
|-----------|--------------------------------|----------------|
| What is evolved | Code / programs | Task answers (test-time) |
| Where routing happens | Which LLM proposes the next **mutation** | Which LLM refines the next **candidate answer** |
| Routing signal | Cost-aware **UCB bandit** (reward = fitness gain, radius ∝ $) | **Per-problem difficulty percentile** from confidence/diversity |
| Fitness | Graded `evaluate.py` score | **Verifier-free** proxy (log-probs / diversity) |
| Durability | Produces a reusable program | Produces one query's answer |
| Kinship | AlphaEvolve family | RSA / self-consistency family |

Both are concrete engineering answers to *"spend the expensive model only where it earns its cost"* — ShinkaEvolve at the **operator** layer, Squeeze-Evolve at the **per-instance difficulty** layer.

## Connections

- [sources/shinkaevolve](shinkaevolve.md) — sibling cost-aware multi-model design, at the mutation-operator layer
- [sources/alphaevolve](alphaevolve.md) — shares the OpenEvolve reference; the code-evolution cousin
- [sources/autoreason](autoreason.md) — also an inference-time, per-query self-improvement loop; comparison/consensus-style signal rather than graded reward
- [concepts/evolutionary-optimization](../concepts/evolutionary-optimization.md) — population-based search with LLM operators; adds a cost-routing axis
- [concepts/feedback-signals](../concepts/feedback-signals.md) — confidence/diversity as a zero-cost, verifier-free fitness proxy
- [wiki/experiment.md](../experiment.md) — cost-aware model routing; here keyed on per-instance difficulty rather than optimized per-stage

---
title: "Squeeze-Evolve: Verifier-Free Evolutionary Test-Time Scaling with Multi-Model Routing"
type: source
tags: [evolution, test-time-scaling, cost-aware, multi-model-routing, verifier-free, inference-time, colm-2026, rsa]
sources: [squeeze-evolve]
url: https://github.com/squeeze-evolve/squeeze-evolve
code: https://github.com/squeeze-evolve/squeeze-evolve
venue: COLM 2026
last_updated: 2026-08-01
---

# Squeeze-Evolve

**Repo**: [github.com/squeeze-evolve/squeeze-evolve](https://github.com/squeeze-evolve/squeeze-evolve) (COLM 2026)

## One-line summary

A **verifier-free evolutionary test-time-scaling** framework: it runs an evolutionary answer-refinement loop over a population of candidate solutions and **routes each refinement step to a cheap or expensive model based on per-problem difficulty**, estimated with zero-cost confidence/diversity proxies — hitting equivalent-or-better accuracy at a fraction of the inference cost.

## What makes it distinct

Two ideas the wiki hasn't seen combined:

1. **Verifier-free fitness.** No ground-truth checker and no learned reward model. Fitness is a **zero-cost proxy computed from the generation itself**: *group confidence* (token log-probabilities, requires `prompt_logprobs: true`) or *answer diversity* (count of unique answers in a group). This directly answers the failure mode of [sources/interaction-trajectory-mining](interaction-trajectory-mining.md), where an *offline reward model* was too weak a signal — Squeeze-Evolve sidesteps the reward model entirely and reads difficulty off the model's own uncertainty.
2. **Cost-aware model routing by per-instance difficulty.** Unlike uniform test-time scaling (one big model for every problem), it splits candidates into tiers by adaptive per-problem percentile thresholds and sends **hard groups to expensive models, easy groups to cheap ones** — spending compute where it changes the answer.

## Core loop — fitness-based routing

Population-based, five stages per iteration:

```
Loop 0 (init):  expensive model generates N candidates per problem
Loops 1..T:
  1. Score      → group confidence (logprobs) or answer diversity  [~free]
  2. Select     → group candidates into subsets of size K (uniform or fitness-weighted)
  3. Route      → split groups into N tiers by per-problem percentile thresholds;
                  tier → model tier (expensive ↔ hard, cheap ↔ easy)
  4. Recombine  → each model refines its groups in parallel;
                  full-consensus groups use lightweight aggregation (majority vote)
  5. Update     → replace or accumulate candidates for the next loop
```

Single integrated orchestration over one shared population (not co-evolution — all model tiers work the same population). Models are ordered cheapest→most-expensive; the `confidence_percentiles` list defines the N−1 routing boundaries for N models.

## What is optimized / what is not

**Optimized**: the *answer* to each problem, at **inference time** — plus the *allocation of models* across the population.

**Not optimized**: no weights, no prompts, no harness code. This is a **test-time / per-query** loop, on the same time axis as [sources/autoreason](autoreason.md)'s per-query tournament rather than the deployment- or training-time loops that dominate the wiki. Where AutoReason improves one output via an incumbent/adversarial/synthesis tournament with blind judges, Squeeze-Evolve improves a *population* of outputs via confidence-routed recombination — and its "judge" is free (self-confidence) rather than an LLM panel.

## Cost-aware multi-model — relation to ShinkaEvolve and AlphaEvolve

Cost-aware multi-model orchestration now appears at three layers of the evolutionary stack in this wiki:

| System | Where the model choice is made | Signal driving it |
|--------|-------------------------------|-------------------|
| [AlphaEvolve](alphaevolve.md) | Which model *mutates* code (Flash for breadth, Pro for depth) | Fixed role split (exploration vs exploitation) |
| [ShinkaEvolve](shinkaevolve.md) | Which model *mutates* code, per proposal | Cost-aware **UCB bandit** (confidence radius inflated by $ per call) |
| **Squeeze-Evolve** | Which model *refines the solution*, per problem instance | **Per-instance difficulty** (confidence / diversity percentile) |

AlphaEvolve and ShinkaEvolve route the *operator* (who proposes the next code edit); Squeeze-Evolve routes at the *solution/inference* layer (who produces the next answer), keyed on a per-problem difficulty estimate rather than a running bandit. All three are engineering answers to the same cost/capability trade-off that recurs in [wiki/experiment.md](../experiment.md)'s `[prompt, model]` framing.

## Prior work it credits

- **RSA (Recursive Self-Aggregation)** — the evolutionary aggregation loop and recombination prompts are inspired by RSA.
- **[OpenEvolve](coral.md)** — referenced as related evolutionary-coding infrastructure (also the CORAL baseline in this wiki).

The README does not position against [AlphaEvolve](alphaevolve.md), [ShinkaEvolve](shinkaevolve.md), or [GEPA](optimize-anything.md), though it notes evolutionary inference has shown "strong test-time scaling."

## Configuration & terminology

Key YAML knobs:

| Param | Meaning |
|-------|---------|
| `k` | group size |
| `population` | candidates per problem at loop 0 |
| `loops` | number of evolutionary iterations |
| `confidence_percentiles` | N−1 routing thresholds for N models (e.g. `[50.0]` for a 2-model system) |
| `fitness` | `confidence` or `diversity` |
| `selection` | `uniform` or `weighted` (fitness-proportional) |
| `update` | `replace` or `accumulate` |
| `lite_fraction` | easiest groups sent to lightweight majority-vote aggregation |
| `recombination` | `aggregate` (task-aware) or `synthesize` |

## Benchmarks & execution

- Datasets: **AIME 2025, HMMT 2025, GPQA-Diamond** (hard reasoning). Ready-to-run: `bash benchmarks/aime25/run.sh`.
- CLI: `squeeze-evolve-client --config benchmarks/aime25/configs/example.yaml --input data/aime25/test.parquet --n-problems 5`.
- Deployments: integrated as a **NVIDIA Dynamo** component (`dynamo.squeeze_evolve`); **Claude Code plugin** via `/plugin install squeeze-evolve@squeeze-evolve`.
- The README reports cost/accuracy *scaling curves* (`assets/scaling_final_v2_dark.svg`) rather than single headline accuracy numbers — the claim is Pareto (equal-or-better accuracy, fraction of the cost), not a point gain.

## Positioning in this wiki

Squeeze-Evolve is the wiki's clearest example of **cost-aware inference-time evolution**: population-based test-time scaling where the lever is *which model handles which problem*, not *which mutation to make*. It complements the training/deployment-time evolutionary systems and pairs naturally with [ShinkaEvolve](shinkaevolve.md) — the two together show cost-aware model selection is a portable idea at both the operator layer (Shinka) and the solution layer (Squeeze).

## Connections

- [sources/shinkaevolve](shinkaevolve.md) — cost-aware model selection at the *mutation-operator* layer (UCB bandit); Squeeze-Evolve is the *solution/inference* counterpart
- [sources/alphaevolve](alphaevolve.md) — Flash/Pro breadth/depth split is the fixed-role ancestor of difficulty-based routing
- [sources/autoreason](autoreason.md) — the other per-query, inference-time loop; tournament judging vs verifier-free self-confidence
- [sources/interaction-trajectory-mining](interaction-trajectory-mining.md) — its offline-reward-model bottleneck is exactly what verifier-free confidence fitness avoids
- [concepts/evolutionary-optimization](../concepts/evolutionary-optimization.md) — population + selection + recombination, at inference time
- [concepts/feedback-signals](../concepts/feedback-signals.md) — confidence/diversity as a **zero-cost, verifier-free fitness proxy**
- [wiki/experiment.md](../experiment.md) — per-instance cost-aware model routing echoes the `[prompt, model]` cost/capability trade-off

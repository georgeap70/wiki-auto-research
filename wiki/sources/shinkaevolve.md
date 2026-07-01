---
title: ShinkaEvolve
type: source
tags: [evolution, code-optimization, ensemble-llm, sakana-ai, sample-efficient, open-source, bandit, novelty-rejection]
sources: [shinkaevolve]
last_updated: 2026-07-01
---

# ShinkaEvolve

**Paper**: [ShinkaEvolve: Towards Open-Ended and Sample-Efficient Program Evolution](https://arxiv.org/abs/2509.19349) (arXiv:2509.19349, September 2025)
**Authors**: Robert Tjarko Lange, Yuki Imajuku, Edoardo Cetin (Sakana AI)
**Code**: [github.com/SakanaAI/ShinkaEvolve](https://github.com/SakanaAI/ShinkaEvolve)

"Shinka" (進化) is Japanese for *evolution*.

## One-line summary

Open-source, sample-efficient counterpart to [sources/alphaevolve](alphaevolve.md): a population-based program-evolution framework where an ensemble of LLMs mutates code, with three specific mechanisms — adaptive parent sampling, code-novelty rejection sampling, and a **cost-aware UCB bandit over the LLM ensemble** — that cut sample counts by orders of magnitude on tasks like circle packing (150 samples to a new SOTA).

## Why it matters

[sources/alphaevolve](alphaevolve.md) proved the LLM-driven evolutionary-code paradigm can rewrite production infrastructure and beat 56-year-old algorithmic records, but the system is closed. ShinkaEvolve is the first serious open reproduction that:

1. Publishes source, configs, and reproducible variants.
2. Adds explicit **sample-efficiency** machinery — the AlphaEvolve blog reports headline wins but doesn't discuss cost per sample; ShinkaEvolve names it as a first-class objective.
3. Bakes in an **LLM ensemble bandit with cost awareness** — the ensemble includes premium (Gemini 3.1-Pro / GPT-5) and cheap (Gemini 3-Flash / GPT-5-mini) tier models, and the bandit trades off per-sample dollar cost against expected mutation quality.

## Architecture

```
                ┌──── islands (subpopulations) ─────┐
                │  ┌──── archive ────┐              │
                │  │ elites + novel   │             │
                │  │ solutions        │             │
                │  └──────────────────┘             │
                │        ↓                          │
                │  parent sampling                  │
                │  (weighted / power_law / beam)    │
                │        ↓                          │
                │  LLM ensemble bandit ── UCB w/ cost coef
                │  (Gemini 3-Flash, Gemini 3.1-Pro,│
                │   GPT-5-mini, GPT-5 tier)         │
                │        ↓                          │
                │  patch generation (diff / full /  │
                │                    cross)         │
                │        ↓                          │
                │  code-novelty rejection sampling  │
                │        ↓                          │
                │  evaluate.py → score              │
                │        ↓                          │
                │  archive replacement              │
                │  (fitness or crowding)            │
                └───────────────────────────────────┘
             ← periodic migration between islands →
```

## The three innovations

### 1. Adaptive parent sampling

Configurable strategies balance exploitation of top solutions against exploration of the archive:

| Strategy | Behavior |
|----------|----------|
| `weighted` | Sample proportional to fitness with tunable temperature |
| `power_law` | Alpha exponent controls how sharply top solutions dominate |
| `beam_search` | Keep top-K and branch each generation |

Combined with a tunable **archive exploitation ratio** trading elite reuse against draws from the diverse archive.

### 2. Code-novelty rejection sampling

Before evaluating a proposed patch, ShinkaEvolve compares it against the archive on a code-similarity metric. Near-duplicates are **rejected without being evaluated**, saving the expensive scoring call. This is the sample-efficiency lever most directly responsible for the "150 samples for circle packing SOTA" result.

Conceptually related to the diversity-preservation mechanisms in [sources/asi-evolve](asi-evolve.md) (Novelty Check filters near-duplicate motivations) and [sources/group-evolve](group-evolve.md) (performance-novelty selection), but applied at the code-syntactic level and used to **skip evaluations** rather than just to *rank* them.

### 3. Cost-aware UCB bandit over LLM ensemble

The default ensemble mixes tiers and providers — Gemini 3-Flash / Gemini 3.1-Pro / GPT-5-mini / GPT-5. Each proposed mutation picks a model via **Upper Confidence Bound** where the reward is expected fitness gain and the confidence radius is inflated by the model's **per-call dollar cost**. Cheap models get more shots per unit of cost; premium models get pulled only when their expected gain justifies the price.

This is a direct engineering answer to the multi-model choice problem discussed in [wiki/experiment.md](../experiment.md) — but at the *evolutionary operator* level, not at the *deployed skill* level.

## Patch types

Three mutation kinds, sampled probabilistically per proposal:

| Type | What it does |
|------|--------------|
| `diff` | Minimal patch — cheapest, best for local hill-climbing |
| `full` | Full-file rewrite — best for structural changes |
| `cross` | Crossover-style combination of two parents |

## Evaluated tasks

| Task | Result |
|------|--------|
| Circle packing | **New SOTA in only 150 samples** |
| AIME (math reasoning) | Consistent improvements — details in paper |
| ALE-Bench (competitive programming) | Consistent improvements — details in paper |
| Novel mixture-of-experts loss discovery | Discovered non-trivial new loss formulations |

The four-task spread is deliberate breadth: pure math (circle packing), math-reasoning-plus-code (AIME), competitive programming (ALE-Bench), and ML architecture (MoE losses).

## Execution / infrastructure

- **CLI**: `shinka_launch variant=circle_packing_example`
- **Python API**: `ShinkaEvolveRunner` with config objects
- **Agent Skills**: CLI-friendly wrappers for agent integration
- **Distributed evaluation** via SLURM for scale
- Required files per variant: `evaluate.py` (scoring) and `initial.py` (seed program)
- **Islands** (subpopulations) with periodic migration for diversity — a classic evolutionary-computing pattern

## What is being optimized vs not

**Optimized**: the program (`initial.py` → improved versions).

**Not optimized**: the ShinkaEvolve control loop itself, or the LLM weights. Frozen models, fixed loop. Like [sources/alphaevolve](alphaevolve.md), it is a **solution-space optimizer** in [concepts/harness-optimization](../concepts/harness-optimization.md) terms — the harness is the framework itself.

## Comparison to AlphaEvolve

| Dimension | AlphaEvolve | ShinkaEvolve |
|-----------|-------------|--------------|
| Source | Closed, proprietary | **Open source (Apache-ish)** |
| Ensemble | Gemini Flash + Gemini Pro | 4-model default: Gemini 3-Flash / 3.1-Pro / GPT-5-mini / GPT-5 |
| Model selection | Not disclosed | **Cost-aware UCB bandit** |
| Novelty | Not disclosed | **Code-similarity rejection sampling** (skips eval, not just rank) |
| Population structure | Not disclosed | **Islands + archive** |
| Sample efficiency | Not the headline metric | **First-class objective** — 150 samples for circle packing SOTA |
| Notable result | Borg heuristic, 4×4 complex matmul in 48 mults | Circle packing SOTA, novel MoE losses |
| Deployment | Production inside Google | Research + community |

## Positioning in this wiki

ShinkaEvolve is the **most reusable open substrate** in the LLM-evolutionary-code family. It sits alongside:

- [sources/alphaevolve](alphaevolve.md) — the closed, higher-scale reference point
- [sources/optimize-anything](optimize-anything.md) — GEPA, similar Pareto + LLM-operator ideas but framed for prompt/harness optimization
- [sources/evoforge](evoforge.md) — smaller-scale population harness evolution
- [sources/evo](evo.md) — tree-search autoresearch orchestrator with a similar "frontier strategy = selection operator" separation

If you want to actually *run* an AlphaEvolve-style loop on your own problem today, ShinkaEvolve is currently the closest match on the shelf.

## Connections

- [sources/alphaevolve](alphaevolve.md) — direct inspiration; ShinkaEvolve's paper cites it explicitly
- [sources/asi-evolve](asi-evolve.md) — parent sampling policies (UCB1, MAP-Elites) and novelty checks map onto ShinkaEvolve's mechanisms one-for-one
- [sources/optimize-anything](optimize-anything.md) — GEPA's Pareto pruning is a natural upgrade to fitness-only archive replacement
- [concepts/evolutionary-optimization](../concepts/evolutionary-optimization.md) — LLM-as-operator, ensemble selection, novelty-driven diversity
- [concepts/feedback-signals](../concepts/feedback-signals.md) — `evaluate.py` is a scalar fitness signal; extension to richer diagnostics is an obvious next step
- [wiki/experiment.md](../experiment.md) — the cost-aware bandit over models resonates with the final-solution "single-loop GEPA over [prompt, model]" framing, but at the *operator* layer rather than the *deployed-skill* layer

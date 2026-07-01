---
title: AlphaEvolve
type: source
tags: [evolution, code-optimization, ensemble-llm, deepmind, scientific-discovery, matmul, funsearch-successor]
sources: [alphaevolve]
last_updated: 2026-07-01
---

# AlphaEvolve

**Paper**: [AlphaEvolve: A coding agent for scientific and algorithmic discovery](https://arxiv.org/abs/2506.13131) (arXiv:2506.13131, June 2025)
**Authors**: Alexander Novikov, Ngân Vũ, Marvin Eisenberger, Emilien Dupont, Po-Sen Huang, Adam Zsolt Wagner, et al. (Google DeepMind)
**Blog**: [AlphaEvolve — a Gemini-powered coding agent for designing advanced algorithms](https://deepmind.google/discover/blog/alphaevolve-a-gemini-powered-coding-agent-for-designing-advanced-algorithms/)

## One-line summary

An evolutionary coding agent that orchestrates an ensemble of Gemini models (Flash for breadth, Pro for depth) to iteratively mutate whole codebases under automated evaluators — production-deployed inside Google (Borg scheduler, TPU circuits, Gemini training kernels) and used to break Strassen's 56-year-old record for 4×4 complex matrix multiplication.

## Core loop

```
prompt sampler ── assembles context from parents + evaluator history
       ↓
LLM ensemble ── Gemini Flash (breadth) + Gemini Pro (depth) propose edits
       ↓
automated evaluators ── run and score the mutated program
       ↓
selection ── winners feed the next generation's prompt sampler
```

Feedback signal: **automated evaluators** producing quantitative correctness + quality scores. One or more evaluators can be configured per task, giving multi-objective pressure natively.

## What is optimized

**Entire codebases**, not single functions. This is the headline distinction from FunSearch (see below) — AlphaEvolve can evolve large multi-file programs, not just a single scoring routine.

## What is NOT optimized

The base LLMs (Gemini Flash / Pro) — they are used as frozen mutation operators. The evolutionary loop lives entirely at the *code* layer. In [concepts/harness-optimization](../concepts/harness-optimization.md) terms, AlphaEvolve is a **solution-space optimizer**, not a harness optimizer — the "harness" here is the AlphaEvolve pipeline itself, which is fixed.

## Novel contributions vs FunSearch

FunSearch (DeepMind, 2023) demonstrated LLM-driven evolutionary search discovering *provably correct* mathematical knowledge, but was constrained to **single-function** proposals. AlphaEvolve generalizes on three axes:

| Axis | FunSearch | AlphaEvolve |
|------|-----------|-------------|
| Unit of mutation | Single function | Entire codebase / multi-file program |
| LLM role | One model as proposer | Ensemble (Flash for exploration, Pro for exploitation) |
| Evaluators | Single scoring function | One or more automated evaluators, multi-objective possible |
| Deployment | Research demo | Production inside Google — data centers, hardware, training |

## Concrete results

### Google infrastructure (deployed in production)

- **Borg data-center scheduler**: discovered a heuristic that "continuously recovers, on average, 0.7% of Google's worldwide compute resources" — in production for over a year.
- **TPU circuit design**: proposed a Verilog rewrite of an arithmetic circuit for matrix multiplication, removing unnecessary bits while preserving functional equivalence — integrated into upcoming TPUs.
- **Gemini training kernel**: sped up a critical kernel by **23%**, yielding a **1% reduction in Gemini's overall training time** — a huge dollar figure at Google's scale.
- **FlashAttention**: **up to 32.5%** speedup for FlashAttention implementations.

### Algorithmic / mathematical discovery

- **4×4 complex matrix multiplication in 48 scalar multiplications** — the first improvement over Strassen's 1969 algorithm in this setting in 56 years.
- Tested on **50+ open mathematical problems**: rediscovered SOTA in ~75% of cases; **improved the previously best-known solution in 20%**.
- **Kissing number in 11 dimensions**: established a new lower bound.

## Why this belongs in this wiki

AlphaEvolve is the current high-water mark for the "LLMs as evolutionary mutation operators over code" thread that runs through:

- [sources/optimize-anything](optimize-anything.md) — GEPA, the same primitive at prompt/small-code scale
- [sources/asi-evolve](asi-evolve.md) — ASI-Evolve applies the same loop to ML research nodes
- [sources/shinkaevolve](shinkaevolve.md) — the open-source, sample-efficient sibling from Sakana AI
- [sources/evoforge](evoforge.md), [sources/coral](coral.md), [sources/group-evolve](group-evolve.md) — variations on population dynamics

Where AlphaEvolve differs from most sources here is **scale and stakes**: it is not a research prototype claiming a few points on a benchmark — it is a code-mutation loop that permanently changed how one of the world's largest compute fleets is scheduled.

## Loop-structure taxonomy

Under [concepts/self-improvement-loop](../concepts/self-improvement-loop.md):

- **Population-based**, not single-agent self-edit.
- **Rich (structured) feedback** via evaluators, though the paper's abstract stops short of describing failure traces the way [concepts/feedback-signals](../concepts/feedback-signals.md) analyzes them for other systems.
- **Gating**: automated-evaluator scores drive selection; no human-in-the-loop review reported for the production wins.

In CORAL's [Stage 1 / Stage 2 / Stage 3](coral.md) autonomy taxonomy, CORAL explicitly places AlphaEvolve at **Stage 1 — structured evolution with fixed external rules** (external algorithm controls selection). See [concepts/evolutionary-optimization](../concepts/evolutionary-optimization.md).

## What is not disclosed in public materials

- Population size, exact prompt-sampler heuristics, evaluator API, whether patches are full-file rewrites or diffs.
- Whether it uses ASI-style rich diagnostic feedback beyond scalar scores.
- Code is **not released** — AlphaEvolve is a proprietary Google system; open-source approximations are [sources/shinkaevolve](shinkaevolve.md) and OpenEvolve.

## Connections to other work

- [sources/shinkaevolve](shinkaevolve.md) — open-source sample-efficient sibling; explicitly cites AlphaEvolve as inspiration
- [sources/optimize-anything](optimize-anything.md) — GEPA's Pareto selection is a natural next step for multi-evaluator AlphaEvolve
- [sources/coral](coral.md) — CORAL taxonomizes AlphaEvolve as Stage 1 structured evolution
- [sources/asi-evolve](asi-evolve.md) — evolutionary loop, but the "code" being evolved is ML research designs
- [concepts/evolutionary-optimization](../concepts/evolutionary-optimization.md) — population + LLM operators + evaluators
- [concepts/feedback-signals](../concepts/feedback-signals.md) — one-or-more automated evaluators as the fitness signal

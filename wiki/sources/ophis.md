---
title: "OPHIS: Mechanistic Auto-Research (Observation → Problem → Hypothesis → Intervention → Speed-up)"
type: source
tags: [autoresearch, mechanistic, training-dynamics, non-llm, non-evolutionary, causal, metacircle, grokking]
sources: [ophis]
url: https://meta-circle.com/blog/ophis-a-new-paradigm-for-autoresearch
org: MetaCircle (corresponding author Ziming Liu)
published: 2026-07
last_updated: 2026-08-01
---

# OPHIS

**Blog**: [OPHIS: A New Paradigm for AutoResearch](https://meta-circle.com/blog/ophis-a-new-paradigm-for-autoresearch) (MetaCircle, July 2026; corresponding author **Ziming Liu**)

Builds on MetaCircle's prior *Omnigrok* (ICLR Spotlight 2023) on grokking — which took ~6 months of manual analysis to yield 2 training tricks, a process OPHIS aims to automate and generalize.

## One-line summary

The wiki's **contrarian auto-research entry**: OPHIS derives training-optimization interventions from **causal mechanistic understanding of internal training dynamics** — explicitly **no LLM and no evolutionary search**. It measures ~6,000 tensor-level dynamics of a training run, localizes bottlenecks, forms a mechanistic hypothesis, and derives a targeted intervention — "understand *why*, then intervene," instead of remixing internet priors or hill-climbing over past experiments.

## The OPHIS loop

Five stages, **single-agent and autonomous** (human-in-the-loop only for architecture design):

```
Observation   → measure ~6,000 tensor-level training-dynamics quantities
                (norms, entropy-like measures, parameter/activation statistics)
Problem       → localize the bottleneck from the internal observables
Hypothesis    → mechanistically explain the observed phenomenon
Intervention  → derive a targeted modification from the hypothesis (not trial-and-error)
Speed-up      → validate the acceleration against baseline
```

Self-correction handles failed hypotheses and unexpected artifacts. Because interventions are *derived* rather than *searched*, candidates are generated "almost instantly" (vs. ~10 minutes for a strong LLM per candidate in their comparison).

## What is optimized / what is not

**Optimized**: training speed / efficiency of a fixed architecture — a *"Training Copilot."* The artifact changed is the training recipe (interventions on the optimization process), derived from mechanism.

**Not optimized**: the architecture (human-designed, fixed for now); no model weights are the *target* (they are the *observable*); no code is evolved. There is **no LLM anywhere in the loop** — the paper frames itself as "orthogonal to the mainstream LLM approach."

## The feedback signal is the headline difference

Where the rest of the wiki argues *rich execution traces beat scalar reward* ([concepts/feedback-signals](../concepts/feedback-signals.md)), OPHIS goes a level deeper: its signal is the **internal state of the system being optimized** — thousands of curves of the model's own training dynamics. This is the richest, most "white-box" feedback source in the wiki:

| Feedback richness | Example |
|-------------------|---------|
| Scalar reward | pass rate, loss value |
| Execution traces | [Meta-Harness](meta-harness.md) (10M-token traces), [HALO](halo.md) (OpenTelemetry) |
| **Internal training dynamics** | **OPHIS (~6,000 tensor-level observables of the run itself)** |

The intervention is *causally attributable* to observed dynamics — closer to mechanistic interpretability applied to optimization than to black-box search. It even yields **curiosity-driven discovery**: OPHIS surfaced a novel "forking" failure mode (severe train/val separation under specific architectural conditions), localized it to a specific module using ~900 observation curves, and derived a targeted ablation — discovery, not just optimization.

## OPHIS's own Stage 1/2/3 taxonomy — and a naming collision

OPHIS frames auto-research along a **depth-of-causal-understanding** axis:

| OPHIS stage | Approach | Limitation OPHIS names | Wiki analog |
|-------------|----------|------------------------|-------------|
| Stage 1 | Internet prior only — recombine known tricks | Can only remix; can't answer "*why* should the LR increase?" | [Karpathy-style autoresearch](autoresearch-vs-hpo.md) |
| Stage 2 | RSI — statistical memory of past experiments | Remembers *what* worked, not *why*; no causal model | recursive self-improvement lineage ([STOP](stop.md), [DGM](dgm.md)) |
| Stage 3 | **Mechanistic reasoning from internal dynamics** | — (OPHIS's claim) | **OPHIS** |

> ⚠️ **Do not conflate with CORAL's Stage 1/2/3.** [CORAL](coral.md)'s taxonomy (in [concepts/evolutionary-optimization](../concepts/evolutionary-optimization.md)) measures **autonomy of the evolutionary controller** (structured evolution → agent autonomy → open frontiers). OPHIS's stages measure **depth of causal understanding**. Same numbers, orthogonal axes.

## Concrete results

**Grokking (modular addition)** — baseline 95% test accuracy at ~1,250 steps:

| Method | Tricks tested | Substantial improvement | Failure rate |
|--------|--------------:|------------------------:|-------------:|
| OPHIS | 350 | **72.9%** | 13.7% |
| LLM (full observable set) | 76 | 57.9% | 42.1% |
| LLM (OPHIS-like subspace) | 30 | 70.0% | 30.0% |

**NanoGPT (language-model training)** — on an RSI-optimized baseline where [Karpathy's autoresearch](autoresearch-vs-hpo.md) achieved only 0.001 (within noise): OPHIS reduced validation **BPB from 0.9340967 → 0.9318420** (−0.0022547 = **7.43 standard deviations**), testing 376 interventions with ≥3σ improvements using only **13% of available observables**. Single H200 GPU; **10 repeated evaluations per candidate** to separate real gains from kernel noise.

## Gating / safety angle

Two gates, both distinctive:

- **Mechanistic plausibility filter** — candidate interventions are screened against the mechanistic hypothesis before evaluation, not sampled blindly.
- **Variance / stability gate** — 10 repeated evals per candidate distinguish genuine gains from noise (kin to [evolve-the-harness](evolve-the-harness.md)'s noise-aware promotion and [Evo](evo.md)'s repeated-trial discipline). Some high-mean interventions were unstable; the authors flag optimizing for variance alongside mean as future work.

There is also an implicit safety argument against **[reward hacking](../concepts/regression-gating.md)**: [STOP](stop.md) and [DGM](dgm.md) game mis-specified objectives precisely *because* they search blindly over a proxy. Deriving interventions from a causal model of *why* something helps is a structurally different stance — you optimize the mechanism, not a hackable scalar. (OPHIS does not claim to *solve* reward hacking; but "understand then intervene" attacks its root cause rather than gating its symptoms.)

## Positioning in this wiki

OPHIS is the **foil to almost everything else here**. The wiki's dominant paradigms are LLM-as-mutation-operator ([AlphaEvolve](alphaevolve.md), [ShinkaEvolve](shinkaevolve.md), [GEPA](optimize-anything.md)), LLM-agent-owns-the-loop ([autoresearch](autoresearch-vs-hpo.md), [Meta-Harness](meta-harness.md)), and self-modifying code ([STOP](stop.md)→[DGM](dgm.md)→[Hyperagents](hyperagents.md)). OPHIS argues all of these are "superficial" — they lack a causal model of *why* an intervention works — and demonstrates a non-LLM, non-evolutionary alternative that beats an LLM baseline on its chosen tasks. Whether "mechanistic understanding" generalizes beyond training-dynamics optimization to open-ended agent design is the open question it leaves.

## Limitations & future work (author-stated)

- Currently **architecture-fixed** (a Training Copilot), not architecture discovery.
- Needs "richer observations, stronger abstractions, and more comprehensive models of learning dynamics" to scale.
- Some high-performing interventions were unstable → optimize for variance, not just mean.
- Long-term vision: discover *new architectures* via mechanistic learning-dynamics theory.

## Connections

- [sources/autoresearch-vs-hpo](autoresearch-vs-hpo.md) — Karpathy-style autoresearch = OPHIS's "Stage 1"; the NanoGPT comparison is head-to-head against it
- [sources/stop](stop.md), [sources/dgm](dgm.md) — recursive self-improvement / self-modifying code = OPHIS's "Stage 2"; also the reward-hacking cases its causal stance argues against
- [sources/meta-harness](meta-harness.md), [sources/halo](halo.md) — rich-trace feedback, one level less deep than internal training dynamics
- [concepts/feedback-signals](../concepts/feedback-signals.md) — internal training-dynamics observables as the richest, white-box end of the feedback spectrum
- [concepts/self-improvement-loop](../concepts/self-improvement-loop.md) — a mechanistic, *non-search* instantiation of the measure→propose→gate loop
- [concepts/regression-gating](../concepts/regression-gating.md) — mechanistic-plausibility + variance gating; "understand then intervene" vs. blind proxy search

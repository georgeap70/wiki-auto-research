---
title: OPHIS (Mechanistic Auto-Research)
type: source
tags: [autoresearch, mechanistic, non-llm, non-evolutionary, training-dynamics, grokking, nanogpt, metacircle]
sources: [ophis]
url: https://meta-circle.com/blog/ophis-a-new-paradigm-for-autoresearch
org: MetaCircle (corresponding author Ziming Liu)
published: 2026-07
last_updated: 2026-07-31
---

# OPHIS

**Blog**: [OPHIS: A New Paradigm for AutoResearch](https://meta-circle.com/blog/ophis-a-new-paradigm-for-autoresearch) (MetaCircle, July 2026)
**Corresponding author**: Ziming Liu (builds on *Omnigrok*, ICLR 2023 Spotlight)

## One-line summary

The **contrarian entry** in this wiki: an auto-research system that derives training optimizations from **causal mechanistic insight** — measuring ~6,000 tensor-level training-dynamics observables and reasoning about *why* an intervention should work — **without using LLMs and without evolutionary search**. Its acronym is its loop: **O**bservation → **P**roblem → **H**ypothesis → **I**ntervention → **S**peed-up.

## Why it belongs here (as a foil)

Almost every other source in this wiki is either an LLM-driven proposer (GEPA, AutoResearch, Meta-Harness) or an evolutionary population (AlphaEvolve, ShinkaEvolve, EvoForge). OPHIS explicitly rejects both, arguing they operate *superficially*: they "remix internet knowledge or remember past successes without understanding **why** interventions work" — a system that can't "properly answer questions like… 'why should learning rates increase?'" is pattern-matching, not doing science. OPHIS is the wiki's clearest statement of the **mechanistic-understanding** alternative to search-based self-improvement.

## The five-stage loop

| Stage | What happens |
|-------|--------------|
| **Observation** | Measure ~6,000 tensor-level training-dynamics quantities (norms, entropy-like measures, parameter/activation statistics) |
| **Problem** | Identify bottlenecks from the internal observable dynamics |
| **Hypothesis** | Mechanistically *explain* the observed phenomenon |
| **Intervention** | Derive a targeted modification **from the hypothesis** (not trial-and-error) |
| **Speed-up** | Validate acceleration against baseline |

Single-agent, autonomous (human-in-the-loop only for architecture design), with self-correction when hypotheses fail or unexpected artifacts appear. Candidate interventions are generated "almost instantly" versus ~10 minutes for a GPT-5.5-based approach.

## OPHIS's own Stage 1/2/3 taxonomy

OPHIS frames auto-research on a ladder of **causal depth** — this is a *different taxonomy* from [sources/coral](coral.md)'s Stage 1/2/3 **autonomy** ladder; do not conflate the two (see [concepts/self-improvement-loop](../concepts/self-improvement-loop.md)):

| OPHIS stage | Basis | Exemplar |
|-------------|-------|----------|
| **Stage 1** | Internet prior only; limited to recombination | Karpathy-style autoresearch ([sources/autoresearch-vs-hpo](autoresearch-vs-hpo.md)) |
| **Stage 2** | Statistical memory of past experiments; no causal model | RSI-style loops (memory of what worked) |
| **Stage 3** | Mechanistic reasoning from internal training dynamics | **OPHIS** |

> ⚠️ **Taxonomy collision**: CORAL's Stage 1 = "structured evolution with fixed external rules" (an *autonomy* claim); OPHIS's Stage 1 = "recombines an internet prior" (a *causal-depth* claim). They rank different things. AlphaEvolve is CORAL-Stage-1; Karpathy autoresearch is OPHIS-Stage-1 — these are not the same "Stage 1."

## Feedback signal — a new richest tier

OPHIS extends the [feedback-signals](../concepts/feedback-signals.md) spectrum past execution traces. Where [sources/meta-harness](meta-harness.md) reads source + execution traces and [sources/halo](halo.md) reads production OpenTelemetry traces, OPHIS reads the **internal state of the optimization process itself** — thousands of per-tensor dynamics curves — and uses them to *localize* problems to specific modules. This is diagnostic feedback one layer deeper than any trace: not "what the program did" but "what the weights are doing while training."

- **Multi-criterion**: mean validation metric, stability, and a standard-deviation threshold.
- **Gating/safety**: candidate interventions are filtered through *mechanistic plausibility* before testing; failures are diagnosed, not just discarded.
- **Noise handling**: 10 repeated evaluations per candidate on a single H200 to separate real gains from kernel noise.

## Concrete results

### Grokking (modular addition)

Baseline: 95% test accuracy at ~1,250 steps.

| Method | Tricks tested | Substantial improvement | Failure rate |
|--------|--------------:|------------------------:|-------------:|
| **OPHIS** | 350 | **72.9%** | **13.7%** |
| LLM (full observable space) | 76 | 57.9% | 42.1% |
| LLM (OPHIS-like subspace) | 30 | 70.0% | 30.0% |

OPHIS both tests far more interventions and fails far less often — the mechanistic prior raises the hit rate, not just the volume. Notably, giving an LLM the *same* restricted observable subspace closes much of the gap (70.0% vs 72.9%), suggesting the observable design is a large part of the win.

### NanoGPT (language-model optimization)

- Baseline: an already **RSI-optimized** implementation; a Karpathy-autoresearch pass yielded only 0.001 improvement (within noise).
- OPHIS: validation BPB **0.9340967 → 0.9318420** (−0.0022547 = **7.43σ**), testing 376 interventions with ≥3σ improvements using only **13%** of available observables.

## Discovery beyond optimization

OPHIS also surfaced a novel phenomenon it named **"forking"** — a severe train/validation separation under specific architectural conditions — and used mechanistic analysis of ~900 observation curves to localize it to a specific module and derive a targeted ablation. The authors frame this as **curiosity-driven scientific discovery**, not merely goal-directed optimization.

## Limitations

- Scope is **architecture-fixed** today — a "Training Copilot," not an architecture searcher.
- Some high-performing proposals were **unstable**; the next iteration should optimize for variance alongside the mean.
- Needs "richer observations, stronger abstractions, and more comprehensive models of learning dynamics" to reach the long-term goal of discovering *new architectures* from mechanistic learning theory.

## Positioning in this wiki

- Direct foil to [sources/autoresearch-vs-hpo](autoresearch-vs-hpo.md): Karpathy autoresearch is explicitly OPHIS's **Stage 1**.
- Complementary to the LLM/evolutionary mainstream — its mechanistic observables are a candidate **richer feedback source** any of those loops could consume (imagine feeding ~6,000 training-dynamics signals as ASI to a GEPA-style proposer).
- Provenance ties to *Omnigrok*: the original grokking analysis took ~6 months of manual work to yield 2 tricks; OPHIS automates and generalizes that.

## Connections

- [sources/autoresearch-vs-hpo](autoresearch-vs-hpo.md) — the Karpathy-style loop OPHIS classifies as Stage 1 and outperforms on NanoGPT
- [sources/coral](coral.md) — a *different* Stage 1/2/3 taxonomy (autonomy, not causal depth); flagged to prevent confusion
- [sources/meta-harness](meta-harness.md), [sources/halo](halo.md) — trace-based rich feedback; OPHIS reads a deeper internal signal (training dynamics)
- [concepts/feedback-signals](../concepts/feedback-signals.md) — mechanistic internal-dynamics observables as a new richest tier
- [concepts/self-improvement-loop](../concepts/self-improvement-loop.md) — a non-LLM, non-evolutionary loop variant (O→P→H→I→S)

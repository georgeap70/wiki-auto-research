---
title: "Self-Harness: Harnesses That Improve Themselves"
type: source
tags: [harness-optimization, self-improvement-loop, weakness-mining, regression-gating, model-specific, terminal-bench]
sources: [self-harness]
url: https://arxiv.org/abs/2606.09498
code: none
authors: Hangfan Zhang, Shao Zhang, Kangcong Li, Chen Zhang, Yang Chen, Yiqun Zhang, Lei Bai, Shuyue Hu
arxiv: 2606.09498
last_updated: 2026-07-10
---

# Self-Harness (arXiv 2606.09498)

## Summary

Self-Harness is a self-improvement paradigm in which an LLM agent **autonomously refines its own operating harness** — with no human engineers and no external optimizer agent in the loop. It is the cleanest published instantiation of the [Meta-Harness](meta-harness.md) pattern as a self-contained, single-model loop, and it contributes one sharp empirical claim: **the right harness edit is model-specific**. Different base models fail in different ways, so the loop turns *each model's own* weaknesses into concrete, executable harness changes rather than piling on generic instructions.

## The Three-Stage Loop

1. **Weakness Mining** — identify model-specific failure patterns by clustering execution traces from runs of the current harness. This is the [failure-mining](../concepts/feedback-signals.md) step, targeted at *this model's* recurring errors.
2. **Harness Proposal** — the model proposes *diverse yet minimal* modifications aimed at the mined weaknesses. Minimality is deliberate: small, targeted edits rather than sweeping rewrites.
3. **Proposal Validation** — regression-test each candidate; accept only changes that are non-detrimental. This is a [regression gate](../concepts/regression-gating.md) on held-in/held-out splits, so an edit that fixes new failures but breaks prior successes is rejected.

The loop runs entirely from the agent's own execution feedback — success/failure on tasks — with no human intervention after initialization.

## What Is Optimized / Feedback / Gating

| Axis | Self-Harness |
|------|--------------|
| Optimization target | The operating harness (scaffolding around a fixed model) |
| Feedback signal | Execution success/failure + clustered failure traces (weakness mining) |
| Loop structure | Single-model self-edit (no separate optimizer agent) |
| Gating | Regression validation on held-in/held-out; accept only non-detrimental edits |
| Human involvement | None after initialization |

## Results (Terminal-Bench-2.0)

Task-completion pass rate, three diverse base models, harness-only changes (no weight updates):

| Model | Baseline | Self-Harness | Gain |
|-------|----------|--------------|------|
| MiniMax M2.5 | 40.5% | 61.9% | +21.4pp |
| Qwen3.5-35B-A3B | 23.8% | 38.1% | +14.3pp |
| GLM-5 | 42.9% | 57.1% | +14.2pp |

The gains hold across a frontier model, a mid-size MoE, and a strong open model — evidence that harness slack exists regardless of base capability.

## Why It Matters

- **Model-specific harnesses**: the headline conceptual contribution. Prior harness-optimizers ([auto-harness](auto-harness.md), [Meta-Harness](meta-harness.md)) implicitly optimize *a* harness; Self-Harness argues the optimum is *per-model*, because the failure distribution is per-model. This complicates the "one good harness transfers everywhere" story — see [evolve-the-harness](evolve-the-harness.md), which finds code mechanisms transfer across model families but prompt playbooks do not.
- **Self-contained loop**: no external optimizer agent. This is a stricter form of autonomy than the [optimizer/target separation](../concepts/self-improvement-loop.md) pattern ([HALO](halo.md), [SkillOpt](skillopt.md), [RLM-GEPA](rlm-gepa.md)) — the model that runs the tasks is also the one that edits the harness.
- Called out by name in [Lilian Weng's harness-engineering survey](weng-harness-blog.md) as the exemplar of the "self-improving harness" category (alongside STOP).

## Connections

- [concepts/harness-optimization](../concepts/harness-optimization.md) — a self-contained instantiation; adds the model-specific-edit claim
- [concepts/self-improvement-loop](../concepts/self-improvement-loop.md) — three-stage weakness-mining loop, single-model self-edit
- [concepts/feedback-signals](../concepts/feedback-signals.md) — weakness mining = model-specific failure clustering
- [concepts/regression-gating](../concepts/regression-gating.md) — non-detrimental-edit validation on held-in/held-out
- [sources/meta-harness](meta-harness.md) — the pattern Self-Harness self-contains
- [sources/evolve-the-harness](evolve-the-harness.md) — a concurrent Meta-Harness application; converging and contrasting findings on transfer
- [sources/weng-harness-blog](weng-harness-blog.md) — survey that names Self-Harness as the self-improving-harness exemplar

---
title: "AutoResearch vs Classical Hyperparameter Tuning (Weco AI)"
type: source
tags: [hyperparameter-optimization, agentic-search, code-space, architectural-change]
sources: [autoresearch-vs-hyperparams]
url: https://www.weco.ai/blog/autoresearch-vs-classical-hpo
code:
  - https://github.com/WecoAI/weco-cli
  - https://github.com/karpathy/autoresearch
authors: Zhengyao Jiang, Bingchen Zhao
org: Weco AI
published: 2026-04-02
last_updated: 2026-04-04
---

# AutoResearch vs Classical Hyperparameter Tuning

## Summary

Weco AI benchmarks their **AutoResearch** agent (based on Claude Code) against Optuna's TPE optimizer on NanoChat model training — using identical compute budgets on H100 GPUs. The agent wins on sample efficiency, cost, and generalization.

## Setup

- Task: optimize NanoChat training (a small language model training run)
- Baseline: Optuna TPE — a state-of-the-art classical hyperparameter optimizer
- Agentic approach: Claude Code agent operating over the **full code space**, not just a predefined parameter grid
- Identical H100 GPU compute budgets

## Why the Agent Wins

**Classical HPO** is bounded by its search space: it can only tune parameters it was explicitly given (learning rate, batch size, etc.). It cannot question the architecture or restructure the training loop.

**AutoResearch** operates over the full code:
1. First, identifies *which parameters actually matter* before tuning them (avoids wasted search)
2. Tunes those parameters efficiently
3. Eventually **pivots to structural architectural changes** — things classical HPO cannot express at all

The agent's domain knowledge acts as **implicit regularization**, preventing overfitting to proxy metrics that classical HPO is susceptible to.

## Key Result

The agent discovers architectural improvements that are **outside the search space of any hyperparameter optimizer**. This demonstrates that the natural ceiling for classical HPO is the boundary of the predefined parameter grid — agents have no such ceiling.

## Framing: Optimization as Self-Improvement

This is [self-improvement](../concepts/self-improvement-loop.md) at the **model-training level**. The agent:
- Measures: training loss, validation metrics
- Fails: identifies underperforming configurations
- Proposes: changes ranging from hyperparameter tweaks to structural rewrites
- Gates: validates on held-out eval before committing

The agent expands its own search space mid-run — a form of meta-adaptation that classical tools cannot do.

## Connections

- entities/optimize-anything generalizes this idea: if it's text and it's measurable, it can be optimized
- [concepts/feedback-signals](../concepts/feedback-signals.md) — agent uses training logs + domain knowledge, not just scalar loss
- [concepts/self-improvement-loop](../concepts/self-improvement-loop.md) — demonstrates the loop at the ML training layer
- entities/evox — also expands search space dynamically, but via evolutionary strategy switching rather than domain reasoning

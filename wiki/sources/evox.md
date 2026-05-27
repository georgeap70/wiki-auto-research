---
title: "EvoX: Letting AI Evolve Its Own Evolution Process (UC Berkeley Sky Lab)"
type: source
tags: [meta-evolution, evolutionary-optimization, strategy-evolution, competitive-programming]
sources: [skydicover]
url: https://skydiscover-ai.github.io/blog-evox.html
code: https://github.com/skydiscover-ai/skydiscover
authors: Shu Liu, Shubham Agarwal, Monishwaran Maheswaran, et al.
org: UC Berkeley Sky Lab / Bespoke Labs
last_updated: 2026-04-04
---

# EvoX (UC Berkeley Sky Lab)

## Summary

EvoX introduces **meta-evolution**: the optimization strategy itself is evolvable, not just the solutions. A two-level loop runs simultaneously — an outer loop evolves *how* candidates are generated; an inner loop generates candidates using the current strategy.

## The Core Idea

Most evolutionary and LLM-based optimizers fix their search strategy in advance: always do greedy refinement, or always do uniform sampling, or always do UCB. EvoX asks: what if the *strategy* were also subject to evolutionary pressure?

```
outer loop: evolve the strategy (which search operator to use)
  inner loop: generate candidates using current strategy
```

The outer loop monitors:
- Population diversity
- Stagnation (no improvement over N iterations)
- Distribution of current candidates

And updates the strategy accordingly, switching among:
- Uniform sampling (exploration)
- Greedy refinement (exploitation)
- Multi-objective combination
- UCB-based balancing

## Why This Matters

Fixed strategies have known failure modes:
- Greedy refinement → local optima
- Random sampling → slow convergence
- UCB → sensitive to noise

By making the strategy evolvable, EvoX can **escape local optima** by switching to an exploratory strategy precisely when stagnation is detected — a form of adaptive meta-cognition.

## Results

- **+34% median improvement** on 172 competitive programming problems
- State-of-the-art results for under $5 on select benchmarks
- Across ~200 tasks tested

## Relation to Self-Improvement

EvoX is the most meta of the approaches in this collection. It demonstrates **[[concepts/self-improvement-loop|self-improvement]] at the algorithm level**:

| Level | What improves | System |
|-------|--------------|--------|
| Task solution | The answer | Agent0 |
| Code policy | Constraint wrapper | AutoHarness |
| Harness | Prompt + tools | auto-harness, Meta-Harness |
| Architecture | Agent code | optimize_anything |
| **Optimization algorithm** | **How search is done** | **EvoX** |

## Connections

- [[concepts/evolutionary-optimization]] — EvoX extends classical evolutionary ideas with LLM operators and meta-strategy evolution
- [[sources/optimize-anything]] — both use evolutionary frameworks with LLMs; EvoX focuses on strategy evolution, GEPA on Pareto + ASI
- [[concepts/self-improvement-loop]] — the meta-loop in EvoX is itself an instance of the core improvement loop
- [[sources/autoresearch-vs-hpo]] — both demonstrate that dynamic search space expansion beats fixed strategies

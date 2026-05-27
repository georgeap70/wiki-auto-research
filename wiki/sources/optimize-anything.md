---
title: "optimize_anything: A Universal API for Optimizing Any Text Parameter (GEPA)"
type: source
tags: [universal-optimization, ASI, pareto, GEPA, evolutionary, code-evolution]
sources: [optimize-anything]
url: https://gepa-ai.github.io/gepa/blog/2026/02/18/introducing-optimize-anything/
code: https://github.com/gepa-ai/gepa
authors: Lakshya A. Agrawal, Donghyun Lee, et al. (13 authors)
org: UC Berkeley, Stanford
published: 2026-02-18
last_updated: 2026-04-04
---

# optimize_anything (GEPA)

## Summary

GEPA proposes a universal framework: **if an artifact is representable as text and its performance is measurable, it can be optimized**. This covers code, prompts, agent architectures, configs — anything. The key innovations are Actionable Side Information (ASI) and Pareto-efficient multi-metric search.

## Core Thesis

The "optimize anything" claim is literal:
- Code → optimize for correctness + speed
- Prompts → optimize for task performance
- Agent architecture → optimize for capability
- Training configs → optimize for loss
- Rules / heuristics → optimize for accuracy

All of these share the same API: `optimize(artifact, evaluate, side_info)`.

## Key Innovations

### Actionable Side Information (ASI)
Rich diagnostic feedback — compiler errors, profiler traces, test output, execution logs — is elevated to a **first-class input** to the optimizer, not reduced to a scalar. The proposer LLM reads ASI to understand *why* a candidate failed before proposing the next one.

This formalizes what [[sources/meta-harness]] and [[sources/autoharness-arxiv]] discovered empirically: rich context produces better proposals than scalar reward alone.

### Pareto-Efficient Multi-Metric Search
Rather than collapsing multiple objectives (correctness, speed, cost, robustness) into a weighted scalar, GEPA maintains a **Pareto frontier** of non-dominated candidates. This avoids:
- Averaging away specialization
- Proxy-metric overfitting
- Losing high-value candidates that are excellent on one metric but mediocre on another

### LLMs as Proposers (GEPA = Genetic-Pareto + LLMs)
The underlying framework is genetic/evolutionary, but the mutation and crossover operators are replaced with LLM proposal calls. This gives the system domain knowledge (it doesn't randomly perturb code; it makes semantically informed changes) while retaining the diversity and exploration properties of evolutionary search.

## ARC-AGI Case Study

The most striking result: an entire agent system evolved from a **10-line stub** into a **300+ line architecture** with rule induction, pattern matching, and verification modules.

Starting point:
```python
def solve(grid): return grid  # 10 lines
```

After optimization:
- Rule induction module
- Pattern matching engine  
- Verification and self-correction layer
- ~300 lines of working code

No human wrote the intermediate steps. The optimizer grew the system from near-nothing.

## Connections

- [[concepts/feedback-signals]] — ASI formalizes rich feedback as a first-class concept
- [[concepts/evolutionary-optimization]] — GEPA is built on Pareto + genetic search with LLM operators
- [[sources/meta-harness]] — converging on the same rich-feedback principle from a different direction
- [[sources/autoresearch-vs-hpo]] — also demonstrates optimization over full code space
- [[entities/evox]] — also evolves the optimization process itself; complementary approach

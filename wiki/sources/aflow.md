---
title: "AFlow: Automating Agentic Workflow Generation"
type: source
tags: [workflow-optimization, mcts, code-represented-workflows, operators, cost-efficiency, foundational]
sources: [aflow]
url: https://arxiv.org/abs/2410.10762
code: https://github.com/FoundationAgents/AFlow
authors: Jiayi Zhang, Jinyu Xiang, Zhaoyang Yu, Fengwei Teng, et al.; corresponding Yuyu Luo, Chenglin Wu
org: DeepWisdom (MetaGPT) + HKUST(GZ) + Renmin U. + Nanjing U. + Fudan + KAUST + Mila/UdeM
arxiv: 2410.10762
venue: ICLR 2025 (Oral)
last_updated: 2026-07-10
---

# AFlow: Automating Agentic Workflow Generation (arXiv 2410.10762)

## Summary

AFlow reframes agentic-workflow design as a **search over code-represented workflows** and solves it with **Monte Carlo Tree Search (MCTS)** where *each tree node is a complete workflow*. It is the wiki's canonical example of MCTS-driven harness search, and its headline economic result — a small model running an AFlow-discovered workflow beating GPT-4o at ~4.55% of the cost — is one of the strongest statements of the "optimize the scaffold, not the model" thesis that unifies this wiki (see [harness-optimization](../concepts/harness-optimization.md), [evolve-the-harness](evolve-the-harness.md)).

## Core Method

- **Workflow representation**: LLM-invoking **nodes** (params: model, prompt, temperature, output format) connected by **edges expressed as code** (sequences, conditionals, loops — more expressive than fixed graphs).
- **Operators**: reusable node combinations as building blocks — Generate, Format, Review, Revise, Ensemble, Test, Programmer (+ Custom).
- Search space simplified by fixing model/temperature/format, so search focuses on **prompts + code edges + operator composition**.
- **MCTS variant**, four phases:
  1. **Soft mixed-probability selection** — `P = λ·uniform + (1−λ)·softmax(score)` (α=0.4, λ=0.2); the initial workflow is always retained to escape local optima.
  2. **LLM-based expansion** — an optimizer LLM (Claude-3.5-Sonnet) edits prompts / node connections using the selected workflow's logged experience.
  3. **Execution evaluation** — run the candidate 5× on a validation set for mean/std.
  4. **Experience backpropagation** — record the modification vs. parent and success/failure; propagate score up the tree.

## What Is Optimized / Feedback / Loop / Gating

| Axis | AFlow |
|------|-------|
| Optimization target | The entire agentic workflow (prompts + code edges + operators) |
| Feedback signal | Execution scores **+** tree-structured "experience" (logged modifications, improvements/failures) — richer than a bare scalar |
| Loop structure | Single population evolving via MCTS (up to 20 rounds) |
| Gating | Early stopping when top-k average stalls for n rounds; 5× runs to de-noise; Pareto over performance vs. token cost |

## Results (executors = GPT-4o-mini unless noted)

- **+5.7%** avg over manually designed methods; **+19.5%** over prior automated workflow optimization; **80.3%** average across 6 benchmarks.
- Per-benchmark: HotpotQA 73.5 F1, DROP 80.6 F1, HumanEval 94.7 pass@1, MBPP 83.4 pass@1, GSM8K 93.5, MATH 56.2. Beats [ADAS](adas.md) on MATH and MBPP by ~57%.
- **Cost/capability**: AFlow-found workflows let a smaller model **outperform GPT-4o on specific tasks at 4.55% of GPT-4o's inference cost**.
- Workflows transfer across executor models, though the optimal workflow differs per model.

## Why It Matters

- **MCTS over workflows** is the missing middle between [ADAS](adas.md)'s free-form meta-agent search and flat population evolution — a structured, tree-guided search with explicit experience backpropagation. Compare to [Evo](evo.md)'s tree-search frontier and [AlphaEvolve](alphaevolve.md)'s population.
- The cost result is direct empirical backing for the wiki's [experiment.md](../experiment.md) cost/capability framing and the "mismanaged-geniuses" thesis in [evolve-the-harness](evolve-the-harness.md).

## Connections

- [concepts/evolutionary-optimization](../concepts/evolutionary-optimization.md) — MCTS as a tree-structured population search with experience backpropagation
- [concepts/harness-optimization](../concepts/harness-optimization.md) — the workflow *is* the harness; AFlow optimizes its structure
- [concepts/self-improvement-loop](../concepts/self-improvement-loop.md) — MCTS select→expand→evaluate→backprop is a loop instantiation
- [sources/adas](adas.md) — predecessor (free-form code search); AFlow adds MCTS structure and beats it on MATH/MBPP
- [sources/evo](evo.md) — also tree-search autoresearch, but over committed experiments with configurable frontier strategies
- [sources/weng-harness-blog](weng-harness-blog.md) — cites AFlow under "workflow design" (workflows as LLM-action graphs optimized by MCTS)

---
title: ASI-Evolve — AI Accelerates AI
type: source
tags: [evolutionary, multi-agent, architecture-search, data-curation, rl-algorithm, cognition-base, research-automation]
sources: [asi-evolve]
last_updated: 2026-04-06
---

# ASI-Evolve — AI Accelerates AI

**Paper:** [arXiv 2603.29640](https://arxiv.org/abs/2603.29640) — March 31, 2026
**Code:** [GAIR-NLP/ASI-Evolve](https://github.com/GAIR-NLP/ASI-Evolve)
**Institution:** GAIR-NLP
**Corresponding author:** Pengfei Liu

---

## What Is Being Solved

Existing agentic self-improvement frameworks handle well-scoped tasks with fast feedback (code generation benchmarks, math puzzles). ASI-Evolve targets **long-horizon AI research loops** — the kind that require GPU-hours, open-ended solution spaces, and multi-dimensional experimental results. The paper formalizes this gap via a "Scientific Task Length" (L_task) measure with three axes:

- **C_exec (Execution Cost):** compute, codebase modification, GPU hours required
- **S_space (Search Space Complexity):** breadth of the solution space; task openness
- **D_feedback (Feedback Complexity):** difficulty of extracting actionable signal from noisy multi-metric experimental output

Prior frameworks (FunSearch, AlphaEvolve, OpenEvolve) operate at low C_exec and low D_feedback. ASI-Evolve is designed for the high-high-high regime.

---

## Core Loop: Learn → Design → Experiment → Analyze

Three agent roles collaborate in a four-stage cycle:

```
Learn → Design (Researcher) → Experiment (Engineer) → Analyze (Analyzer)
```

**Learn:** Sample n nodes from the solution database. Retrieve relevant entries from the [[concepts/knowledge-accumulation|Cognition Base]] via semantic search (embedding-indexed). Inject retrieved context into the Researcher's prompt.

**Design (Researcher agent):** Generate a complete candidate program plus a natural-language *motivation* explaining why this design is promising. Supports full-code generation and diff-based editing modes.

**Experiment (Engineer agent):** Execute the candidate via the user-specified evaluation procedure. Return structured metrics and a primary scalar score. Applies three safeguards:
- Static check agent (complexity/structure verification)
- Debug agent (runtime error auto-fixing)
- Novelty check (filter candidates whose motivation is near-duplicate of existing ones)

**Analyze (Analyzer agent):** Receive the full program + complete experimental output (logs, metrics, traces). Distill into a compact, decision-oriented report capturing causal analysis — *why* this result happened, what it implies for future designs. Store the complete node (motivation, code, results, analysis, score) in the database.

The Analyzer is the key innovation here: rather than feeding raw experimental output downstream (risking context explosion), it interprets multi-dimensional feedback before storage. See [[concepts/feedback-signals]].

---

## Cognition Base

A textual repository of human prior knowledge — task-relevant heuristics, known pitfalls, design principles from domain literature. Examples:
- Architecture search: ~150 entries from 100 papers on linear attention and SSMs
- Data curation: quality issue profiles from sampled data
- RL algorithm design: 10 papers covering variance reduction post-GRPO

Retrieved via embedding-based semantic search over candidate motivations. Allows the system to benefit from existing human expertise even in novel domains.

---

## What Is Being Optimized

ASI-Evolve targets three foundational AI development domains simultaneously:

1. **Neural architecture design** — discovering efficient linear attention architectures (search space: variants of DeltaNet)
2. **Pretraining data curation pipelines** — category-specific text cleaning strategies for 672B token corpora
3. **Reinforcement learning algorithms** — redesigning GRPO's advantage allocation and gradient computation

Plus a transfer application to **drug-target interaction (DTI) prediction** (DrugBAN architecture).

---

## Database Sampling Policies

Multiple strategies for selecting parent nodes to build upon:
- **UCB1** (Upper Confidence Bound) — empirically fastest; balances exploitation vs. exploration
- **Random** — uniform sampling across all nodes
- **Greedy** — always sample highest-scoring node
- **MAP-Elites island algorithm** — maintains diversity across behavioral dimensions; prevents premature convergence

---

## Key Results

### Neural Architecture Design
- 1,773 exploration rounds; 1,350 candidates generated; 105 surpassed DeltaNet in verification
- At 1.3B params / 100B tokens: **+0.97 average benchmark improvement** over DeltaNet
- ~3× the gain of the best recent human-designed improvement (Mamba2: +0.34)
- Pattern in top architectures: adaptive multi-scale routing with dynamic computational budget allocation

### Data Curation
- **+3.96 average benchmark points** over best baseline (Nemotron-CC)
- MMLU: +18.64 points (27.49 → 46.13); CSQA: +18.80; MedQA: +13.48
- Final corpus: 504B tokens (down from 672B — cleaner and smaller)

### RL Algorithm Design
- 300 exploration rounds; 10 outperforming candidates; 3 verified at 14B scale
- vs. GRPO: AMC32 **+12.5**, AIME24 **+11.67**, AIME25 **+9.58**, OlympiadBench **+5.04**, Math500 **+3.6**

Two novel RL algorithms discovered:
- **Pairwise Asymmetric Optimization (Algorithm A):** tanh-normalized pairwise reward differences, asymmetric PPO clipping [ε_down, ε_up], High-Impact Gradient Dropout
- **Budget-Constrained Dynamic Radius (Algorithm B):** percentile normalization, Global Update Budget — trusted radius set inversely proportional to advantage magnitude

### Circle Packing Benchmark
- ASI-Evolve (GPT-5-mini): **2.6360 in 17 steps** — fastest to SOTA vs. OpenEvolve (460 rounds), AlphaEvolve (not reported)

### Drug-Target Interaction (Transfer)
- DrugBAN baseline: AUROC 94.15, F1 86.89
- ASI-Evolve best: AUROC **96.06**, F1 **89.84**

---

## Ablation Findings

- **No Analyzer:** High initial scores, long plateau; sporadic and less reproducible improvements. Raw experimental output becomes noise.
- **No Cognition Base:** Delayed onset, slower early improvement, but still effective eventually. Confirms usability in novel domains without prior literature.

Both components compound; neither alone is sufficient for the full benefit.

---

## Relation to Other Sources

| Axis | ASI-Evolve | Comparable sources |
|------|------------|-------------------|
| What optimized | Data / Architecture / RL algorithms | [[sources/optimize-anything]] (architecture), [[sources/evox]] (search strategy) |
| Loop | Learn-Design-Experiment-Analyze | [[sources/meta-harness]] (trace-informed single-agent) |
| Feedback | Analyzer distillation + benchmark scores | [[sources/meta-harness]] (raw traces), [[sources/optimize-anything]] (ASI) |
| Cognition | Embedding-indexed literature base | [[sources/auto-harness]] (learnings.md) |
| Gating | Novelty check + multi-stage scaling | [[concepts/regression-gating]] |

The most distinctive contribution: treating the **Analyzer as a compression layer** — preventing context explosion in high-D_feedback domains by summarizing experimental results before they enter the next iteration.

---

## Key Terms

- **ASI-Evolve:** the framework
- **Scientific Task Length (L_task):** three-axis complexity measure (C_exec, S_space, D_feedback)
- **Cognition Base:** embedding-indexed prior knowledge repository
- **Researcher / Engineer / Analyzer:** three agent roles
- **Learn–Design–Experiment–Analyze:** the four-stage loop
- **Pairwise Asymmetric Optimization:** novel RL algorithm with asymmetric PPO clipping
- **Budget-Constrained Dynamic Radius:** novel RL algorithm with global update budget
- **High-Impact Gradient Dropout:** masking highest-probability × advantage tokens
- **Novelty check:** motivation-similarity filter against redundant candidates

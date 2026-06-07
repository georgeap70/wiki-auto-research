---
title: Evolutionary Optimization for Agentic Systems
type: concept
tags: [evolution, genetic, pareto, population, meta-evolution, EvoX, GEPA, multi-agent]
sources: [evox, optimize-anything, asi-evolve, coral, deep-research, group-evolve, evoforge, honedhaiku, evo-hq]
last_updated: 2026-06-06
---

# Evolutionary Optimization

Population-based search methods applied to agentic self-improvement. Rather than a single agent iterating on itself, evolutionary approaches maintain a **population of candidates** and apply selection pressure over generations.

## Why Evolutionary Methods?

Single-agent self-edit (like [[sources/auto-harness]]) can get stuck in local optima: the agent makes locally good changes but misses globally better configurations that require intermediate "worse" steps.

Evolutionary methods address this by:
- Maintaining diversity via a population
- Combining good features from multiple candidates (crossover)
- Applying selection pressure at the population level
- Running multiple lines of exploration in parallel

## LLMs as Evolutionary Operators

Classical genetic algorithms use random mutation and structured crossover. In agentic settings, **LLMs replace these random operators**:

| Classical | LLM-based |
|-----------|-----------|
| Random mutation | Semantically informed edit proposals |
| Structured crossover | LLM-synthesized combination of two candidates |
| Fitness function | Task benchmark + [[concepts/feedback-signals|rich diagnostic feedback]] |
| Selection | Pareto frontier ([[sources/optimize-anything]]) or tournament |

LLMs as operators bring domain knowledge to the search — they don't randomly permute code, they understand what changes are likely to improve performance.

## GEPA (Genetic-Pareto + LLMs) — [[sources/optimize-anything]]

GEPA combines:
- **Pareto-efficient multi-metric selection**: maintains a frontier of non-dominated candidates, avoiding scalar aggregation
- **LLM proposal operators**: semantically informed mutations
- **ASI (Actionable Side Information)**: rich diagnostic feedback informs proposal generation

The ARC-AGI case study: a 10-line stub evolved into a 300+ line architecture with rule induction, pattern matching, and verification. The population-based approach allowed the system to grow structurally — individual iterations might degrade one aspect while advancing another, but the Pareto frontier retains all valuable variants.

## ASI-Evolve (Database Sampling Policies) — [[sources/asi-evolve]]

ASI-Evolve uses the same evolutionary insight (population + selection) but applied to a database of research nodes rather than an explicit population. Selection policies determine which prior work the Researcher agent builds upon:

| Policy | Behavior |
|--------|----------|
| UCB1 | Balances exploitation (high score) vs. exploration (uncertainty); empirically fastest |
| Greedy | Always sample the highest-scoring node |
| Random | Uniform sampling across all nodes |
| MAP-Elites | Maintains behavioral diversity across dimensions; prevents premature convergence |

The Novelty Check (filtering near-duplicate motivations) acts as an implicit diversity maintenance mechanism — ensuring the population of attempted designs doesn't collapse onto the same region.

## CORAL (Multi-Agent Co-Evolution) — [[sources/coral]]

CORAL is the most autonomous evolutionary system in this collection: multiple agents run independently, each exploring the solution space using their own strategies, while sharing a common knowledge store. No external algorithm prescribes which candidates to retrieve or evaluate.

This is **Stage 2 Agent Autonomy** in CORAL's taxonomy — contrasted with Stage 1 (structured evolution with fixed external rules):

| Stage | Control | Examples |
|-------|---------|---------|
| Stage 1 — Structured Evolution | External algorithms control search | FunSearch, AlphaEvolve |
| Stage 2 — Agent Autonomy | Agents control their own strategies | **CORAL** |
| Stage 3 — Open Frontiers | Multi-agent organizations, human-agent co-evolution | Research direction |

Multi-agent co-evolution yields gains beyond additional compute: four-agent CORAL outperformed best-of-4 independent single-agent runs on all stress-test tasks. Gains derive from cross-agent knowledge transfer: 36% of four-agent attempts used another agent's commit as parent; cross-agent commits improved at 17% vs. 9% overall improvement rate.

## GEPA for Prompt Optimization — [[sources/deep-research]]

GEPA was introduced by [[sources/optimize-anything]] for architecture and code search. [[sources/deep-research]] applies the same primitive to **system prompt search** across a four-agent Deep Research pipeline:

- Maintains a population of prompt candidates per agent
- Prunes strictly dominated candidates (Pareto pruning)
- Samples with probability proportional to Pareto support (frontier frequency)
- Applies patience limit (2 non-improving rounds) before stopping

Result: GEPA with a custom meta-prompt (0.705) outperforms expert-designed prompts + TextGrad (0.672) on PhD-level CS research queries. This demonstrates GEPA's generality: the same selection mechanism works across code, architecture, and natural-language prompt spaces.

## Group-Evolving Agents (Group as Unit) — [[sources/group-evolve]]

GEA challenges the assumption that the **individual** is the right evolutionary unit. In all prior systems (GEPA, EvoX, ASI-Evolve, CORAL), selection and mutation operate on individual candidates or individual agents. GEA promotes the **group** to be the fundamental unit:

- A group of agents collectively produces experiences that go into a **shared experience pool**
- Offspring agents are initialized from this pool, inheriting the distilled experience of the entire group — not just a single parent branch
- Selection uses a **performance-novelty criterion** that jointly optimizes benchmark score and evolutionary diversity, preventing the group from collapsing onto a homogeneous configuration

This makes experience sharing an explicit architectural choice rather than a side effect of shared memory (as in CORAL). The result:

| Benchmark | GEA | Prior self-evolving SOTA | Delta |
|-----------|-----|--------------------------|-------|
| SWE-bench Verified | 71.0% | 56.7% | +14.3pp |
| Polyglot | 88.3% | 68.3% | +20.0pp |
| Framework bug repair | 1.4 iters | 5 iters | 3.6× faster |

The framework bug repair result is particularly notable: the group's pooled diagnostic experience enables much faster root-cause identification than any single agent's exploration.

## EvoX (Meta-Evolution) — [[sources/evox]]

EvoX adds a second evolutionary loop on top of the standard one:

```
outer loop: evolve the search strategy
  inner loop: generate candidates using current strategy
```

Available strategies include:
- Uniform sampling (exploration)
- Greedy refinement (exploitation)
- Multi-objective combination
- UCB-based balancing

The outer loop monitors stagnation and switches strategies accordingly. This is **meta-evolution**: the algorithm that does the optimizing is itself optimized.

**Key insight**: No single search strategy dominates across all tasks and all phases of optimization. Adaptive strategy selection beats any fixed strategy.

## EvoForge (Population Harness Evolution) — [[sources/evoforge]]

EvoForge applies the hill-climbing harness loop from [[sources/autoagent-kevinrgu]] to an entire population simultaneously. Each agent in the population independently mutates its `agent.py` harness, scores against a benchmark, and keeps improvements. After each round, successful mutations are synthesized into shared learnings.

Key characteristics:
- **Three-tier abstraction**: `evolve.md` (population strategy) → `program.md` (per-agent intent) → `agent.py` (implementation)
- **Feedback**: scalar benchmark score (0–1), identical gating to AutoAgent (KevinRGU)
- **Knowledge sharing**: post-round synthesis, weaker than CORAL's real-time shared memory but more structured than independent runs
- **Result**: 2× Codex CLI, 10× baseline on GPT-5-nano

This is the simplest extension of single-agent harness hill-climbing to the population setting — parallelism for throughput and diversity, without the complex co-evolution dynamics of CORAL or GEA.

## Evo (Tree-Search Autoresearch Orchestrator) — [[sources/evo]]

Evo sits between single-thread hill-climbing and flat population evolution: the orchestrator maintains a **tree of committed experiments** and applies a configurable **frontier strategy** to pick which branch to extend each round.

| Frontier strategy | Behavior |
|-------------------|----------|
| `argmax` | extend the highest-scoring branch |
| `top_k` | round-robin among the K best |
| `epsilon_greedy` | best most of the time, random sometimes |
| `softmax` | sample weighted by score |
| `pareto_per_task` | *"keep specialists the aggregate hides"* — credited to GEPA |

Within a round, parallel subagents run in isolated git worktrees; each picks up shared state (failure traces, annotations, *discarded hypotheses*), forms a hypothesis, edits, and benchmarks. Between rounds, RLM-inspired cross-cutting scan subagents read trace batches and surface gate-failure intersections and shared root causes — feedback compression as a between-round phase.

Key differences from flat population evolution:
- **Tree preserves lineage** — every candidate's ancestry is explicit; the dashboard makes the tree inspectable. Closer in spirit to [[sources/autogenesis]]'s versioned-resource lineage than to EvoForge's flat generations.
- **Frontier strategy = selection operator** — the population analog of "which parents are eligible to mutate" is exposed as a tuning knob, including a `pareto_per_task` variant that brings GEPA's intuition (don't average specialists into mediocrity) into a tree-search context.
- **Discarded-hypothesis storage** — negative results are kept in shared state and read by future subagents; conceptually analogous to [[sources/skillopt]]'s rejected-edit buffer at the granularity of experimental directions.
- **Multi-backend execution** — worktree/pool/ssh/modal/e2b/daytona/aws/azure are packaged into the orchestrator; the same loop runs locally during development and on cloud sandboxes for an overnight run.

## GEPA for Coding Prompts — [[sources/honedhaiku]]

[[sources/deep-research]] showed GEPA generalizes from code to prompt search. HonedHaiku applies the same primitive to **bug-fixing system prompts**:

- Mutation-selection loop: GEPA proposes prompt variants; Agentelo scores against real PR test suites
- Converged in 4 of 20 allocated iterations
- Training diversity: 3-challenge run overfits; 20 challenges across 5 repos generalizes

Key finding — the **Goldilocks band**: GEPA only moves performance in the ~50–70% baseline range. Below 50%, the model can't execute complex prompts. Above 85%, prompts are no longer the bottleneck. This constrains the applicability of prompt optimization as a self-improvement method and is a practical complement to the theoretical analysis in [[concepts/feedback-signals]].

## The Hierarchy of Evolution

| Level | What evolves | System |
|-------|-------------|--------|
| Solutions | Task answers, code | Many systems |
| Operators | How mutations are made | GEPA ([[sources/optimize-anything]]) |
| Strategy | Which operators to use | EvoX ([[sources/evox]]), ASI-Evolve sampling policies |
| Agent coordination | Who knows what; which approach to copy | CORAL ([[sources/coral]]) |
| **Evolutionary unit** | **Individual → group as selection unit** | **GEA ([[sources/group-evolve]])** |
| Harness population | Multiple harnesses evolve in parallel | EvoForge ([[sources/evoforge]]) |
| Experiment tree + frontier | Tree-shaped lineage with configurable selection per round | Evo ([[sources/evo]]) |
| Objectives | What fitness means | Open question |

## Connections

- [[concepts/self-improvement-loop]] — evolutionary loops are a population-based instantiation of the core measure-fail-propose-gate cycle
- [[concepts/harness-optimization]] — EvoForge and HonedHaiku both optimize harnesses; EvoForge at population scale, HonedHaiku at prompt level
- [[concepts/regression-gating]] — Pareto gating replaces scalar threshold gating in evolutionary settings
- [[concepts/feedback-signals]] — ASI is particularly important when LLMs act as proposal operators; HonedHaiku's PR test suites are a high-quality grounded signal
- [[sources/optimize-anything]] — GEPA implementation
- [[sources/evox]] — meta-evolution implementation

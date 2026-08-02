---
title: Evolutionary Optimization for Agentic Systems
type: concept
tags: [evolution, genetic, pareto, population, meta-evolution, EvoX, GEPA, multi-agent, self-modifying-code, mcts, archive]
sources: [evox, optimize-anything, optimize-anything-omni, asi-evolve, coral, deep-research, group-evolve, evoforge, honedhaiku, evo-hq, alphaevolve, shinkaevolve, squeeze-evolve, stop, adas, aflow, dgm, hyperagents]
last_updated: 2026-08-01
---

# Evolutionary Optimization

Population-based search methods applied to agentic self-improvement. Rather than a single agent iterating on itself, evolutionary approaches maintain a **population of candidates** and apply selection pressure over generations.

## Why Evolutionary Methods?

Single-agent self-edit (like [sources/auto-harness](../sources/auto-harness.md)) can get stuck in local optima: the agent makes locally good changes but misses globally better configurations that require intermediate "worse" steps.

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
| Fitness function | Task benchmark + [rich diagnostic feedback](feedback-signals.md) |
| Selection | Pareto frontier ([sources/optimize-anything](../sources/optimize-anything.md)) or tournament |

LLMs as operators bring domain knowledge to the search — they don't randomly permute code, they understand what changes are likely to improve performance.

## GEPA (Genetic-Pareto + LLMs) — [sources/optimize-anything](../sources/optimize-anything.md)

GEPA combines:
- **Pareto-efficient multi-metric selection**: maintains a frontier of non-dominated candidates, avoiding scalar aggregation
- **LLM proposal operators**: semantically informed mutations
- **ASI (Actionable Side Information)**: rich diagnostic feedback informs proposal generation

The ARC-AGI case study: a 10-line stub evolved into a 300+ line architecture with rule induction, pattern matching, and verification. The population-based approach allowed the system to grow structurally — individual iterations might degrade one aspect while advancing another, but the Pareto frontier retains all valuable variants.

## ASI-Evolve (Database Sampling Policies) — [sources/asi-evolve](../sources/asi-evolve.md)

ASI-Evolve uses the same evolutionary insight (population + selection) but applied to a database of research nodes rather than an explicit population. Selection policies determine which prior work the Researcher agent builds upon:

| Policy | Behavior |
|--------|----------|
| UCB1 | Balances exploitation (high score) vs. exploration (uncertainty); empirically fastest |
| Greedy | Always sample the highest-scoring node |
| Random | Uniform sampling across all nodes |
| MAP-Elites | Maintains behavioral diversity across dimensions; prevents premature convergence |

The Novelty Check (filtering near-duplicate motivations) acts as an implicit diversity maintenance mechanism — ensuring the population of attempted designs doesn't collapse onto the same region.

## CORAL (Multi-Agent Co-Evolution) — [sources/coral](../sources/coral.md)

CORAL is the most autonomous evolutionary system in this collection: multiple agents run independently, each exploring the solution space using their own strategies, while sharing a common knowledge store. No external algorithm prescribes which candidates to retrieve or evaluate.

This is **Stage 2 Agent Autonomy** in CORAL's taxonomy — contrasted with Stage 1 (structured evolution with fixed external rules):

| Stage | Control | Examples |
|-------|---------|---------|
| Stage 1 — Structured Evolution | External algorithms control search | FunSearch, [AlphaEvolve](../sources/alphaevolve.md), [ShinkaEvolve](../sources/shinkaevolve.md) |
| Stage 2 — Agent Autonomy | Agents control their own strategies | **CORAL** |
| Stage 3 — Open Frontiers | Multi-agent organizations, human-agent co-evolution | Research direction |

Multi-agent co-evolution yields gains beyond additional compute: four-agent CORAL outperformed best-of-4 independent single-agent runs on all stress-test tasks. Gains derive from cross-agent knowledge transfer: 36% of four-agent attempts used another agent's commit as parent; cross-agent commits improved at 17% vs. 9% overall improvement rate.

## GEPA for Prompt Optimization — [sources/deep-research](../sources/deep-research.md)

GEPA was introduced by [sources/optimize-anything](../sources/optimize-anything.md) for architecture and code search. [sources/deep-research](../sources/deep-research.md) applies the same primitive to **system prompt search** across a four-agent Deep Research pipeline:

- Maintains a population of prompt candidates per agent
- Prunes strictly dominated candidates (Pareto pruning)
- Samples with probability proportional to Pareto support (frontier frequency)
- Applies patience limit (2 non-improving rounds) before stopping

Result: GEPA with a custom meta-prompt (0.705) outperforms expert-designed prompts + TextGrad (0.672) on PhD-level CS research queries. This demonstrates GEPA's generality: the same selection mechanism works across code, architecture, and natural-language prompt spaces.

## Group-Evolving Agents (Group as Unit) — [sources/group-evolve](../sources/group-evolve.md)

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

## EvoX (Meta-Evolution) — [sources/evox](../sources/evox.md)

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

## EvoForge (Population Harness Evolution) — [sources/evoforge](../sources/evoforge.md)

EvoForge applies the hill-climbing harness loop from [sources/autoagent-kevinrgu](../sources/autoagent-kevinrgu.md) to an entire population simultaneously. Each agent in the population independently mutates its `agent.py` harness, scores against a benchmark, and keeps improvements. After each round, successful mutations are synthesized into shared learnings.

Key characteristics:
- **Three-tier abstraction**: `evolve.md` (population strategy) → `program.md` (per-agent intent) → `agent.py` (implementation)
- **Feedback**: scalar benchmark score (0–1), identical gating to AutoAgent (KevinRGU)
- **Knowledge sharing**: post-round synthesis, weaker than CORAL's real-time shared memory but more structured than independent runs
- **Result**: 2× Codex CLI, 10× baseline on GPT-5-nano

This is the simplest extension of single-agent harness hill-climbing to the population setting — parallelism for throughput and diversity, without the complex co-evolution dynamics of CORAL or GEA.

## Evo (Tree-Search Autoresearch Orchestrator) — [sources/evo](../sources/evo.md)

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
- **Tree preserves lineage** — every candidate's ancestry is explicit; the dashboard makes the tree inspectable. Closer in spirit to [sources/autogenesis](../sources/autogenesis.md)'s versioned-resource lineage than to EvoForge's flat generations.
- **Frontier strategy = selection operator** — the population analog of "which parents are eligible to mutate" is exposed as a tuning knob, including a `pareto_per_task` variant that brings GEPA's intuition (don't average specialists into mediocrity) into a tree-search context.
- **Discarded-hypothesis storage** — negative results are kept in shared state and read by future subagents; conceptually analogous to [sources/skillopt](../sources/skillopt.md)'s rejected-edit buffer at the granularity of experimental directions.
- **Multi-backend execution** — worktree/pool/ssh/modal/e2b/daytona/aws/azure are packaged into the orchestrator; the same loop runs locally during development and on cloud sandboxes for an overnight run.

## AlphaEvolve (Whole-Codebase Evolution at Google Scale) — [sources/alphaevolve](../sources/alphaevolve.md)

AlphaEvolve is the FunSearch successor from Google DeepMind: an evolutionary coding agent that evolves **entire codebases** (not just single functions) under an ensemble of Gemini models with automated evaluators. It is the most consequential production deployment of the LLM-as-evolutionary-operator paradigm in this wiki.

Structural features:

| Element | Choice |
|---------|--------|
| Unit of mutation | Whole codebase / multi-file program |
| Ensemble | **Gemini Flash** (breadth / exploration) + **Gemini Pro** (depth / exploitation) |
| Feedback | One or more automated evaluators returning quantitative correctness + quality |
| Selection | Winners feed next generation's prompt-sampler context |
| Autonomy | CORAL Stage 1 — external algorithm controls search |

The **Flash-for-breadth / Pro-for-depth** split is the notable operator-level design choice: fast/cheap models generate volume, capable/slow models refine. This is the evolutionary-operator analog of the cost/capability trade-off that appears repeatedly in this wiki (see [wiki/experiment.md](../experiment.md) and [sources/shinkaevolve](../sources/shinkaevolve.md)'s cost-aware bandit).

Production wins: Borg scheduler heuristic recovering **0.7% of Google's global compute**; Gemini training kernel **+23%** → **1% overall training-time reduction**; FlashAttention **+32.5%**; TPU circuit simplifications. Algorithmic wins: **4×4 complex matmul in 48 scalar multiplications** (first improvement over Strassen in 56 years); new lower bound for kissing number in 11D; SOTA rediscovery or improvement on ~95% of 50+ open math problems tested.

## ShinkaEvolve (Sample-Efficient Open AlphaEvolve) — [sources/shinkaevolve](../sources/shinkaevolve.md)

ShinkaEvolve (Sakana AI) is the open-source, sample-efficient answer to AlphaEvolve. It shares the "population + LLM-ensemble mutation + evaluators" skeleton but adds three explicit sample-efficiency mechanisms:

| Innovation | What it does | Related work here |
|------------|--------------|-------------------|
| **Adaptive parent sampling** | `weighted` / `power_law` / `beam_search` with archive-exploitation ratio | EvoX strategy switching; ASI-Evolve UCB1 |
| **Code-novelty rejection sampling** | Syntactic-similarity check rejects near-duplicates **before evaluation** — saves the scoring call | ASI-Evolve Novelty Check (motivation-level); GEA performance-novelty (rank-level) |
| **Cost-aware UCB bandit** | Chooses among Gemini 3-Flash / 3.1-Pro / GPT-5-mini / GPT-5 by UCB where the confidence radius is inflated by per-call $ cost | Direct engineering answer to the model-selection question in [experiment.md](../experiment.md) |

Population structure: **islands with periodic migration**, an archive using either fitness- or crowding-based replacement, and three patch types (`diff` / `full` / `cross`) sampled probabilistically.

Signature result: **new SOTA circle packing in only 150 samples** — headline for the "sample-efficient" claim. Other tasks: AIME reasoning, ALE-Bench competitive programming, and discovering novel mixture-of-experts loss functions.

The cost-aware bandit is the most portable idea for other systems in this wiki: any evolutionary loop with an LLM-mutation-operator stage can drop this in when using a multi-tier ensemble.

## Squeeze-Evolve (Verifier-Free, Cost-Aware Test-Time Evolution) — [sources/squeeze-evolve](../sources/squeeze-evolve.md)

Squeeze-Evolve moves cost-aware model selection **down one layer** from ShinkaEvolve — from *which model mutates the code* to *which model refines each solution at inference*. It runs a population-based evolutionary loop **at test time** (score → select → route → recombine → update) and routes each candidate group to a cheap or expensive model by **per-problem difficulty**, estimated with a **zero-cost, verifier-free fitness proxy**: group confidence (token log-probabilities) or answer diversity.

Two things make it notable here:

- **Verifier-free fitness.** Unlike every other selection signal in this section (benchmark score, evaluator, grader), Squeeze-Evolve needs no checker and no learned reward model — it reads difficulty off the model's own uncertainty. This is the constructive answer to [interaction-trajectory-mining](../sources/interaction-trajectory-mining.md)'s failure (an offline reward model too weak to curate a skill library): skip the reward model, use self-confidence.
- **Difficulty-keyed routing, not a bandit.** Where [ShinkaEvolve](../sources/shinkaevolve.md) uses a running UCB bandit over operators, Squeeze-Evolve thresholds a *per-instance* difficulty percentile and assigns cheapest→dearest models across the population — spending compute where the answer is still uncertain.

Cost-aware model orchestration now spans three rungs of the evolutionary stack: fixed-role split ([AlphaEvolve](../sources/alphaevolve.md) Flash/Pro), running bandit over operators ([ShinkaEvolve](../sources/shinkaevolve.md)), and per-instance difficulty routing over solutions (Squeeze-Evolve). All three engineer the same cost/capability trade-off central to [wiki/experiment.md](../experiment.md).

## optimize_anything Omni (Portfolio of Whole Optimizers) — [sources/optimize-anything-omni](../sources/optimize-anything-omni.md)

Omni lifts the [EvoX](../sources/evox.md) insight to the top of the stack. EvoX says *no single search strategy dominates*, and evolves the strategy; omni says *no single **optimizer** dominates*, and **portfolios the optimizers themselves**. The three engines it races — [GEPA](../sources/optimize-anything.md), [AutoResearch](../sources/autoresearch-vs-hpo.md), [Meta-Harness](../sources/meta-harness.md) — are three separate wiki systems, now interchangeable behind one contract.

The `omni` schedule is **explore-then-continue**: run all engines in parallel on a fraction of the budget, keep the best candidate, then hand it to a *fresh* optimizer for the rest. On Frontier-CS (10 competitive-programming problems, $20 each) each engine wins ~⅓ of problems unpredictably, yet **every omni composition beats every standalone optimizer** (GEPA 43.8→61.8, AutoResearch 55.4→63.2, Meta-Harness 50.9→59.3). The largest lift lands on the most plateau-prone engine — direct evidence for "a fresh optimizer breaks plateaus." Unlike [MCE](../sources/mce.md) or the [self-modifying-code lineage](#the-self-modifying-code-lineage-stop--adas--dgm--hyperagents), omni does not *rewrite* the optimizer — it *selects and reseeds* among fixed engines (a portfolio/meta-scheduling layer, not self-modification). See [concepts/harness-optimization](harness-optimization.md) for the engines as harness optimizers.

## GEPA for Coding Prompts — [sources/honedhaiku](../sources/honedhaiku.md)

[sources/deep-research](../sources/deep-research.md) showed GEPA generalizes from code to prompt search. HonedHaiku applies the same primitive to **bug-fixing system prompts**:

- Mutation-selection loop: GEPA proposes prompt variants; Agentelo scores against real PR test suites
- Converged in 4 of 20 allocated iterations
- Training diversity: 3-challenge run overfits; 20 challenges across 5 repos generalizes

Key finding — the **Goldilocks band**: GEPA only moves performance in the ~50–70% baseline range. Below 50%, the model can't execute complex prompts. Above 85%, prompts are no longer the bottleneck. This constrains the applicability of prompt optimization as a self-improvement method and is a practical complement to the theoretical analysis in [concepts/feedback-signals](feedback-signals.md).

## The Self-Modifying-Code Lineage (STOP → ADAS → DGM → Hyperagents)

A distinct evolutionary sub-family — predating most of the wiki — evolves *the agent's own code* rather than solutions or prompts. It is the lineage [Weng's survey](../sources/weng-harness-blog.md) traces for recursive self-improvement:

- **[STOP](../sources/stop.md)** (Zelikman et al., 2023) — the earliest. A single "improver" program recursively rewrites *itself* under a scalar meta-utility. Not population-based, but the LM autonomously *rediscovered* evolutionary operators (genetic algorithms, simulated annealing, UCB bandits) as improver designs — and produced the wiki's first documented [reward-hacking](regression-gating.md) episode.
- **[ADAS](../sources/adas.md)** (Hu, Lu, Clune, 2024) — Meta Agent Search: a meta-agent programs *new agents* in code, conditioned on a growing **archive**, with novelty pressure toward "interesting" designs. Makes the search space all of code (Turing-complete).
- **[Darwin Gödel Machine](../sources/dgm.md)** (Zhang et al., Sakana AI, 2025) — agents rewrite their *own* codebase; the Gödel machine's proof requirement is replaced by **empirical validation** on benchmarks. Archive/lineage tree; **parent selection ∝ sigmoid(performance) × 1/(offspring count)** — a clean diversity-preserving rule (compare [GEA](../sources/group-evolve.md)'s performance-novelty selection and [ShinkaEvolve](../sources/shinkaevolve.md)'s novelty rejection). SWE-bench Verified 20→50%, Polyglot 14.2→30.7%; transfers across models and languages.
- **[Hyperagents / DGM-H](../sources/hyperagents.md)** (Zhang et al., Meta+UBC, 2026) — merges task-agent and meta-agent into one self-modifiable program so the *modification procedure itself* is editable, targeting any computable task rather than only coding.

The through-line: the mutation operator is a foundation model editing code, the fitness signal is empirical benchmark score, and diversity is preserved by an archive plus an offspring-penalizing parent-selection rule. This family is where "evolve the harness" and "self-improvement loop" most fully coincide.

## AFlow (MCTS over Code-Represented Workflows) — [sources/aflow](../sources/aflow.md)

AFlow is the wiki's example of **Monte Carlo Tree Search as the evolutionary engine**. Each tree node is a *complete workflow* (code-connected LLM-invoking nodes + operators); the four MCTS phases are soft mixed-probability selection (blend of uniform + score-weighted, always retaining the initial workflow), LLM-based expansion (an optimizer LLM edits prompts/edges), 5×-run execution evaluation, and **experience backpropagation** (logged modification-vs-parent + success/failure propagated up the tree).

This sits between [ADAS](../sources/adas.md)'s free-form meta-agent search and flat population evolution, and is close in spirit to [Evo](../sources/evo.md)'s tree-search frontier — but AFlow's tree is an MCTS search tree with backpropagated experience, where Evo's is a lineage of *committed* experiments with a configurable frontier strategy. Result: 80.3% avg over 6 benchmarks (+19.5% over prior automated workflow optimization) and a small model beating GPT-4o at ~4.55% of the cost.

## The Hierarchy of Evolution

| Level | What evolves | System |
|-------|-------------|--------|
| Solutions | Task answers, code | Many systems |
| Operators | How mutations are made | GEPA ([sources/optimize-anything](../sources/optimize-anything.md)) |
| Strategy | Which operators to use | EvoX ([sources/evox](../sources/evox.md)), ASI-Evolve sampling policies |
| Agent coordination | Who knows what; which approach to copy | CORAL ([sources/coral](../sources/coral.md)) |
| **Evolutionary unit** | **Individual → group as selection unit** | **GEA ([sources/group-evolve](../sources/group-evolve.md))** |
| Agent design (as code) | Whole agents programmed by a meta-agent from an archive | ADAS ([sources/adas](../sources/adas.md)) |
| Workflow (MCTS) | Complete workflows as tree nodes; experience backpropagated | AFlow ([sources/aflow](../sources/aflow.md)) |
| Harness population | Multiple harnesses evolve in parallel | EvoForge ([sources/evoforge](../sources/evoforge.md)) |
| Experiment tree + frontier | Tree-shaped lineage with configurable selection per round | Evo ([sources/evo](../sources/evo.md)) |
| Whole codebase | Multi-file program as the mutation unit | AlphaEvolve ([sources/alphaevolve](../sources/alphaevolve.md)) |
| The agent's own code | Agent rewrites its own harness; archive + offspring-penalized selection | DGM ([sources/dgm](../sources/dgm.md)) |
| The improver / modification procedure | The code that does the improving improves itself | STOP ([sources/stop](../sources/stop.md)), Hyperagents ([sources/hyperagents](../sources/hyperagents.md)) |
| Ensemble + cost-aware bandit | Which LLM operator to invoke, weighted by expected gain / $ | ShinkaEvolve ([sources/shinkaevolve](../sources/shinkaevolve.md)) |
| Ensemble routing by instance difficulty | Which model refines each solution, keyed on confidence/diversity (test-time) | Squeeze-Evolve ([sources/squeeze-evolve](../sources/squeeze-evolve.md)) |
| Portfolio of whole optimizers | Which optimizer *family* to run, and when to reseed a fresh one | omni ([sources/optimize-anything-omni](../sources/optimize-anything-omni.md)) |
| Objectives | What fitness means | Open question |

## Connections

- [concepts/self-improvement-loop](self-improvement-loop.md) — evolutionary loops are a population-based instantiation of the core measure-fail-propose-gate cycle
- [concepts/harness-optimization](harness-optimization.md) — EvoForge and HonedHaiku both optimize harnesses; EvoForge at population scale, HonedHaiku at prompt level
- [concepts/regression-gating](regression-gating.md) — Pareto gating replaces scalar threshold gating in evolutionary settings
- [concepts/feedback-signals](feedback-signals.md) — ASI is particularly important when LLMs act as proposal operators; HonedHaiku's PR test suites are a high-quality grounded signal
- [sources/optimize-anything](../sources/optimize-anything.md) — GEPA implementation
- [sources/evox](../sources/evox.md) — meta-evolution implementation

---
title: Group-Evolving Agents
type: source
tags: [evolution, population, multi-agent, experience-sharing, SWE-bench, coding, open-ended]
sources: [group-evolve]
last_updated: 2026-04-08
---

# Group-Evolving Agents

**Paper:** "Group-Evolving Agents: Open-Ended Self-Improvement via Experience Sharing"
**arXiv:** [2602.04837](https://arxiv.org/abs/2602.04837)
**Project:** https://group-evolving-agents.github.io/
**Authors:** Zhaotian Weng, Antonis Antoniades, Deepak Nathani, Zhen Zhang, Xiao Pu, Xin Eric Wang
**Institution:** UC Santa Barbara

## Core Idea

Most self-improving agent systems treat the **individual agent** as the unit of evolution — a single agent mutates itself across iterations. GEA argues that this wastes exploratory diversity: branches of exploration that don't win are simply discarded, even if they contain useful partial solutions.

GEA's insight: make the **group** the fundamental evolutionary unit. Rather than a tree of individual agents where only the winning branch survives, GEA runs a group of agents collectively, aggregating their diverse experiences into a shared pool that all offspring can draw from.

## Architecture

### Two-Stage Evolution

**Stage 1 — Parent Group Selection**

Selection uses a **performance-novelty criterion** that jointly optimizes:
- *Performance:* immediate task competence (benchmark score)
- *Novelty:* evolutionary diversity relative to the rest of the population

This prevents the group from collapsing onto a single high-performing-but-homogeneous configuration, preserving the diversity that enables long-term gains.

**Stage 2 — Open-Ended Group Evolution**

The selected parent group generates offspring that collectively contribute to a **shared experience pool** containing:
- Tool-use traces from successful and failed attempts
- Code patches and diffs
- Strategy notes and heuristics

Offspring are initialized from this shared pool, meaning each new agent inherits the distilled experience of the entire prior group — not just its own lineage.

### Contrast with Prior Work

| Approach | Evolutionary unit | Experience sharing |
|----------|------------------|--------------------|
| Single-agent self-edit (auto-harness, KevinRGU) | Individual | None |
| Tree-structured evolution (FunSearch, AlphaEvolve) | Individual branch | Only via parent lineage |
| CORAL multi-agent | Individual agents + shared notes | Async writes to shared memory |
| **GEA** | **Group** | **Explicit shared experience pool as evolutionary primitive** |

The key distinction from [[sources/coral]]: CORAL's sharing emerges from agents independently reading/writing a common memory store. GEA makes group membership and experience aggregation **explicit architectural choices**, with selection operating on groups rather than individuals.

## Results

| Benchmark | GEA | Prior SOTA (self-evolving) | Delta |
|-----------|-----|--------------------------|-------|
| SWE-bench Verified | **71.0%** | 56.7% | +14.3pp |
| Polyglot | **88.3%** | 68.3% | +20.0pp |
| Framework bug repair | **1.4 iters** | 5 iters | 3.6× faster |

- Matches human-designed agents with no human intervention
- Transferable across different foundation models (results consistent across LLM backends)

## Key Mechanisms

**Performance-novelty selection**: dual-criterion selection prevents premature convergence, maintaining population diversity through the full run — analogous to MAP-Elites or CORAL's cross-agent diversity, but operating explicitly at group level.

**Shared experience pool as evolutionary primitive**: not a byproduct of parallel execution (like CORAL) but the *mechanism of inheritance* — offspring are born from aggregated group experience, not from a single parent.

**Framework bug repair**: GEA can detect and repair bugs in its own scaffolding (framework-level changes), not just task-solving strategy changes. The 1.4 vs 5 iteration comparison suggests group experience dramatically accelerates diagnosis.

## Connections

- [[concepts/evolutionary-optimization]] — new variant: group-as-unit evolution with performance-novelty selection
- [[concepts/knowledge-accumulation]] — shared experience pool is a form of explicit cross-agent knowledge transfer
- [[concepts/self-improvement-loop]] — open-ended evolution without a fixed stopping condition
- [[sources/coral]] — comparison: CORAL uses async shared memory; GEA uses explicit group pooling
- [[sources/evox]] — comparison: EvoX meta-evolves the search strategy; GEA meta-evolves the unit of selection

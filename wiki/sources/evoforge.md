---
title: EvoForge
type: source
tags: [population, harness-optimization, evolution, meta-agent, parallel, haize-labs]
sources: [evoforge]
last_updated: 2026-04-22
---

# EvoForge

**Repo**: [github.com/haizelabs/EvoForge](https://github.com/haizelabs/EvoForge)  
**Author**: Leonard Tang (Haize Labs)

## One-line summary

Evolves a *population* of agent harnesses in parallel — each agent hill-climbs its own `agent.py`, guided by shared `program.md` and `evolve.md` directives, until the population converges on a high-performing harness.

## Core idea

EvoForge stacks three levels of abstraction:

| Layer | File | Who controls it | What it does |
|-------|------|-----------------|--------------|
| Population strategy | `evolve.md` | Human | How many agents, diversity targets, stopping criteria |
| Agent intent | `program.md` | Human (or meta-agent) | Per-agent mutation strategy and goals |
| Harness implementation | `agent.py` | Meta-agent auto-edits | The actual runnable code |

The design principle: *"Program the God of Agents, not the agents directly."* Humans write `evolve.md` to steer the population. A meta-agent reads `program.md` and rewrites `agent.py`. The loop then runs.

## Optimization loop

```
1. Initialize population from evolve.md
2. For each agent in parallel:
   a. Meta-agent proposes mutation to agent.py (guided by program.md)
   b. Run benchmark on the mutated harness
   c. If score improves → keep; else → discard
3. Synthesize knowledge from successful trials across the population
4. Repeat until convergence
```

This is the same hill-climbing loop as [[sources/autoagent-kevinrgu]], but run across a population simultaneously. Knowledge synthesis after each round allows agents to learn from each other's successful mutations — a weaker form of the cross-agent sharing seen in [[sources/coral]].

## What is being optimized

System prompt, tools, configuration, and agent orchestration — the full harness. Score metric is a 0.0–1.0 scalar from benchmark task test suites. Identical gating criterion to [[sources/autoagent-kevinrgu]]: *"Keep if better, discard if not."*

## Key results

- Evolved harness: **2× Codex CLI** and **10× baseline** on GPT-5-nano
- Consistent improvement across all population members (no regression in the published run)

## Connections to other work

- [[sources/autoagent-kevinrgu]] — same `program.md → agent.py` architecture; EvoForge adds population dimension
- [[sources/coral]] — both evolve agent populations; CORAL shares full memory; EvoForge shares synthesized learnings
- [[sources/group-evolve]] — both use population-level parallelism; GEA uses shared experience pool + performance-novelty selection; EvoForge uses scalar score + knowledge synthesis
- [[concepts/harness-optimization]] — EvoForge is harness optimization at population scale
- [[concepts/evolutionary-optimization]] — population + selection is the evolutionary primitive

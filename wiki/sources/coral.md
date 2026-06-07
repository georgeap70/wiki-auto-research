---
title: CORAL — Autonomous Multi-Agent Evolution for Open-Ended Discovery
type: source
tags: [multi-agent, evolutionary, shared-memory, open-ended, co-evolution, heartbeat, emergent-behavior]
sources: [coral]
last_updated: 2026-04-06
---

# CORAL — Autonomous Multi-Agent Evolution for Open-Ended Discovery

**Paper:** [arXiv 2604.01658](https://arxiv.org/pdf/2604.01658) — April 2, 2026
**Code:** [Human-Agent-Society/CORAL](https://github.com/Human-Agent-Society/CORAL)
**Project:** [human-agent-society.github.io/CORAL](https://human-agent-society.github.io/CORAL/)
**Institutions:** MIT, NUS, Stanford, Meta Superintelligence Lab

---

## What Is Being Solved

Existing LLM-based evolutionary systems (FunSearch, AlphaEvolve, ShinkaEvolve) use **fixed heuristics** — the external algorithm controls which solutions to retrieve, when to evaluate, and what knowledge to retain. This limits agent autonomy, prevents sustained knowledge accumulation, and breaks down on truly open-ended problems without predetermined stopping criteria. CORAL asks: can agents run the full evolutionary search themselves while still achieving coordinated, reliable improvement?

---

## Evolutionary Stage Taxonomy

CORAL introduces a three-stage taxonomy of autonomous discovery systems:

| Stage | Description | Examples |
|-------|-------------|---------|
| **Stage 1 — Structured Evolution** | External algorithms control search; agents execute but don't direct | FunSearch, AlphaEvolve, ShinkaEvolve |
| **Stage 2 — Agent Autonomy** | Agents control their own strategies, accumulate knowledge, distill skills | **CORAL** |
| **Stage 3 — Open Frontiers** | Human-agent co-evolution, multi-agent organizations, open-ended evolution | Research direction |

---

## Architecture

### Infrastructure Components

**Task and Evaluation Layer:** User defines what to optimize (codebase, config, description) and how to measure it (grader function returning scalar score + textual feedback).

**Manager:** Handles agent lifecycle — spawning, health monitoring, workspace setup, action validation, communication coordination, feedback delivery.

**Agent Pool:** Multiple agents run in parallel in isolated git worktrees. No predefined roles, no hard-coded communication structures. Agents follow a shared loop but pursue independent strategies.

**Shared Persistent Memory (three layers):**
- `Attempts/` — one JSON per evaluated commit (agent ID, score, status, parent hash, timestamp, grader feedback); complete historical record
- `Notes/` — markdown files with YAML frontmatter; agents freely organize into subdirectories; special dirs: `_synthesis/`, `_connections.md`, `_open-questions.md`; agents read, write, and reorganize freely
- `Skills/` — reusable procedures with natural-language description (SKILL.md) + executable artifacts; standardized via `skill_creator` guide

Access via symlinks from isolated agent worktrees to centralized `.coral/public/`, enabling concurrent reads/writes without locking.

See [concepts/knowledge-accumulation](../concepts/knowledge-accumulation.md).

### Per-Agent Loop

1. **Plan:** run `coral log`, inspect top solutions (`coral show <hash>`), read notes and skills from colleagues
2. **Edit:** make focused modifications with clear intent
3. **Evaluate:** run `coral eval -m "description"` (stages, commits, grades, records atomically)
4. **Reflect:** update shared notes, contribute new skills
5. **Iterate:** navigate prior attempts, identify patterns

Agent instruction: "You are fully autonomous. Do not ask for permission."

### Heartbeat Mechanism (Three Trigger Types)

Prevents fixation — a core failure mode of self-improvement loops. See [concepts/self-improvement-loop](../concepts/self-improvement-loop.md).

| Trigger | Frequency | Purpose |
|---------|-----------|---------|
| **Per-Iteration Reflection** | Every 1 evaluation | Anchor findings; examine surprises; analyze root causes; plan next experiment |
| **Periodic Consolidation** | Every 10 evaluations (global counter) | Create synthesis notes; update connection maps; document contradictions; extract validated techniques into skills |
| **Stagnation-Triggered Pivoting** | After 5 non-improving evaluations | Diagnose performance ceilings; examine top colleagues' attempts; switch to fundamentally different algorithmic approach |

---

## What Is Being Optimized

Open-ended discovery tasks across three domains:

**Mathematical Optimization (6 tasks):** circle packing, signal processing, Erdős minimum overlap problem, MMD-16-2, MMD-14-3, third autocorrelation inequality

**Systems Optimization (5 tasks):** Expert Placement Load Balancing (EPLB), PRISM placement, LLM-SQL optimization, transaction scheduling, cloud transfer (Cloudcast)

**Advanced/Stress Tests:**
- Anthropic Kernel Engineering (VLIW SIMD tree-traversal — minimize CPU cycles)
- Polyominoes Packing (from Frontier-CS benchmark; hardest among 172 problems)

---

## Key Results

### vs. Fixed Evolutionary Baselines (11 standard tasks)
CORAL (single agent) achieves state-of-the-art on **8 of 11 standard tasks** vs. OpenEvolve, ShinkaEvolve, EvoX:
- Improvement rate: **3–10× higher** than fixed evolutionary baselines
- Evaluations to reach quality: **5–20 vs. 60–100** for baselines

### Erdős Minimum Overlap Problem
- CORAL: **0.38089** (outperforms AlphaEvolve's 0.38092)
- **2.5× higher improvement rate, 10× faster evolution, 7× fewer evaluation calls** than OpenEvolve
- Reaches 99% of SOTA in **34 minutes** vs. OpenEvolve's 5.2 hours

### Anthropic Kernel Engineering
- Baseline: 1,363 cycles
- Single agent: **1,350 cycles**
- Four-agent: **1,103 cycles** (18.3% improvement, same wall-clock time)
- 36% of four-agent attempts used another agent's commit as parent
- Cross-agent commits improved at 17% vs. 9% overall improvement rate

### Stanford OpenVaccine (ML Engineering)
- **20.5% improvement** over top human score in **2 minutes and 2 evaluations**

### Polyominoes Packing (Frontier-CS)
- 4-agent CORAL: **84.2 score**
- 4-agent CORAL + web search: **89.4** (vs. previous SOTA 87%)
- 87% of rounds referenced knowledge from other agents

### Multi-Agent vs. Single-Agent (Table 2)
| Task | Single Agent | 4-Agent |
|------|-------------|---------|
| Kernel Engineering | 1,350 cycles | **1,103 cycles** |
| Polyominoes | — | **84.2** (89.4 w/ search) |
| Circle Packing | baseline | +7.90% |
| Signal Processing | baseline | +2.91% |
| Erdős Overlap | baseline | +2.36% |

Four-agent co-evolution **outperformed best-of-4 independent single-agent runs** on all three stress-test tasks — gains are collaborative, not merely additive compute.

### Knowledge Accumulation Impact (Ablation)
| Task | With Notes/Skills | Without |
|------|------------------|---------|
| Kernel Engineering | 1,350 cycles | 1,601 cycles (+18.6% worse) |
| Polyominoes | 80.2 | 77.3 |
| Transaction Scheduling | 4566 | 4444 |

Advanced tasks: 0.55–0.68 knowledge artifacts per attempt; knowledge access associated with **55% improvement rate** (vs. 26% on standard tasks).

---

## Emergent Collaborative Behaviors

CORAL agents develop spontaneous coordination without prescribed protocols:

- **Independent Early Research:** agents maintain separate optimization logs, explore diverse strategies
- **Copycatting:** upon breakthrough, agents rapidly adapt successful techniques from peers
- **Agent Consensus:** at convergence, agents spontaneously form alliances, co-author shared status notes acknowledging dead ends
- **Cross-Referencing:** systematic review of peers' notes; synthesis of hybrid approaches

These behaviors emerge purely from shared memory access and the heartbeat mechanism — no communication protocol is hard-coded.

---

## Relation to Other Sources

| Axis | CORAL | Comparable sources |
|------|-------|-------------------|
| What optimized | Code solutions (math, systems, ML) | [sources/evox](evox.md) (search strategy), [sources/optimize-anything](optimize-anything.md) (architecture) |
| Loop | Autonomous multi-agent co-evolution | [sources/agent0](agent0.md) (two-agent), [sources/evox](evox.md) (meta-evolution) |
| Feedback | Task grader scalar + qualitative shared notes | [sources/meta-harness](meta-harness.md) (execution traces), [sources/asi-evolve](asi-evolve.md) (Analyzer distillation) |
| Knowledge | Attempts/Notes/Skills three-layer memory | [sources/auto-harness](auto-harness.md) (learnings.md), [sources/asi-evolve](asi-evolve.md) (Cognition Base) |
| Gating | Isolated worktrees, health monitoring, stagnation pivot | [concepts/regression-gating](../concepts/regression-gating.md) |

Most distinctive: **agent autonomy over the full search process** (no external algorithm controlling retrieval or selection) combined with **emergent multi-agent coordination** from shared persistent memory.

---

## Key Terms

- **CORAL:** Collaborative Open-ended Research via Autonomous Learners
- **Shared persistent memory:** Attempts/Notes/Skills three-layer store
- **Heartbeat-based interventions:** per-iteration reflection, periodic consolidation, stagnation-triggered pivoting
- **Open-ended discovery:** optimization without fixed endpoints or predetermined fitness landscapes
- **Copycatting:** emergent behavior where agents adopt successful peer strategies
- **Agent Consensus:** emergent behavior where agents co-author notes at convergence
- **Stage 1/2/3 taxonomy:** structured evolution → agent autonomy → open frontiers
- **`coral eval`:** atomic CLI operation: stage/commit/grade/record in one call
- **Stagnation-Triggered Pivoting:** forced algorithmic pivot after 5 non-improving evaluations

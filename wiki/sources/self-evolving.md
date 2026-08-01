---
title: "The What & When of Self-Evolving Agents (Xinming Tu)"
type: source
tags: [taxonomy, survey, self-evolving, framework, overview, what-when-matrix, consolidation]
sources: [self-evolving]
url: http://xinmingtu.cn/blog/2026/self-evolving-agents/
author: Xinming Tu
published: 2026-07-22
last_updated: 2026-07-31
---

# The What & When of Self-Evolving Agents

**Blog**: [The What & When of Self-Evolving Agents](http://xinmingtu.cn/blog/2026/self-evolving-agents/) by **Xinming Tu** (2026-07-22)

Not a system and not a survey — a **taxonomy / analytical framework** for how agents learn from experience. It is the closest external match to this wiki's own organizing question, and it doubles as a lens for placing every other source here on one map.

## The core idea

Learning "must land somewhere, and it must last for some horizon." So classify any self-evolving mechanism by two orthogonal axes:

**What evolves (substrate — increasing durability & generalization cost):**
1. **External Files** — editable artifacts: notes, scripts, skills
2. **Agent Harness** — prompts, tools, workflows, control flow
3. **Model Weights** — parametric knowledge in checkpoints

**When the update persists (horizon):**
1. **Single Session** — within one trajectory (agent-centric: *Intra-Task*, execution feedback)
2. **Across Sessions** — one user/project over time (*Inter-Task*, domain structure)
3. **Across Users** — population-level, benefits everyone (*Inter-Agent*, peer discovery)

The cross-product is a **3×3 matrix** of nine self-evolution patterns.

## The 3×3 matrix (author's placements)

| | **External Files** | **Agent Harness** | **Model Weights** |
|---|---|---|---|
| **Single Session** | autoresearch, AlphaEvolve, MemGPT | Claude Code Dynamic Workflows, Recursive LMs | TTT-Discover, ThetaEvolve |
| **Across Sessions** | Voyager, ACE, Mem0 | Meta-Harness, DSPy, MIPRO, AgentOptimizer, Promptbreeder, GEPA, AFlow, Self-Harness | OpenClaw-RL |
| **Across Users** | EinsteinArena, Agent Skills | Claude Code / Codex defaults, ADAS, Darwin Gödel Machine, Hyperagents | Cursor Tab, Autodata, Self-play |

Feedback sources track the substrate: files use evaluator/test/verification feedback; harness uses aggregate failure logs and execution traces; weights use preference data (accept/reject/edit), verified traces, and reward models.

## Two key dynamics

- **Capability Consolidation Path** — discoveries migrate *upward* through substrates: a temporary task-local artifact → reusable harness logic → internalized model weights. Durability and generalization rise; so does the cost of the update. ("Meta-programming" = compressing repeated execution patterns into reusable DAGs; "consolidation" = promoting a discovery to a more durable substrate.)
- **Recursive self-improvement** is not a separate mechanism — it is "the self-evolving loop applied to AI development itself": agents that generate synthetic pretraining data, build evals, or improve training infra feed the next generation. Tu cites Recursive's automated-AI-research system (agentic search over kernel optimization and model training) as the exemplar.

Design tensions the author flags: portability vs. generalization, reversibility vs. durability, automation vs. governance — and that checkpoint bootstrapping remains "only partially autonomous today." The matrix "shows possibility, not obligation."

## Mapping this wiki onto Tu's matrix

Tu's cells line up cleanly with pages already here — this table is the synthesis contribution of ingesting the source (systems in **bold** have their own wiki page):

| | **External Files** | **Agent Harness** | **Model Weights** |
|---|---|---|---|
| **Single Session** | **[AlphaEvolve](alphaevolve.md)**, **[autoresearch](autoresearch-vs-hpo.md)**, **[Squeeze-Evolve](squeeze-evolve.md)** (per-query), **[AutoReason](autoreason.md)** (per-query) | RLM runtime (**[RLM-GEPA](rlm-gepa.md)**, **[HALO](halo.md)**) | — (no in-session weight learner in the wiki; nearest is **[AgentFlow](agentflow.md)** in-the-flow) |
| **Across Sessions** | `learnings.md` (**[auto-harness](auto-harness.md)**), Cognition Base (**[ASI-Evolve](asi-evolve.md)**), Attempts/Notes/Skills (**[CORAL](coral.md)**), `best_skill.md` (**[SkillOpt](skillopt.md)**), **[WebXSkill](webxskill.md)** | **[Meta-Harness](meta-harness.md)**, **[GEPA](optimize-anything.md)** / **[omni](optimize-anything-omni.md)**, **[HonedHaiku](honedhaiku.md)**, **[EvoForge](evoforge.md)**, **[Evo](evo.md)**, **[ShinkaEvolve](shinkaevolve.md)** | **[AgentFlow](agentflow.md)**, **[TRACE](trace.md)** (LoRA), **[SKILL-RL/SKILL-0](skill-rl-skill0.md)** |
| **Across Users** | shared **Agent Skills** / benchmarks | **[Autogenesis](autogenesis.md)** (protocol for portable self-modification), ADAS / Darwin Gödel Machine (research direction) | population RL from user signal (research direction) |

Two observations the map makes obvious:

1. The wiki is **densest in the "Across Sessions × Harness" cell** — that is exactly the [harness-optimization](../concepts/harness-optimization.md) core, and it is where GEPA/omni, Meta-Harness, Evo, EvoForge, ShinkaEvolve all live.
2. The wiki is **sparse in "Across Users"** — matching [overview.md](../overview.md)'s open questions about population-level flywheels ([sources/coral](coral.md)'s emergent coordination is the nearest thing, and it is still within one run).

## Positioning in this wiki

This page is best read as a **companion to [overview.md](../overview.md)**: overview.md organizes by *what is optimized* and *loop architecture*; Tu adds the orthogonal *persistence horizon* axis (session / project / population), which the wiki had only implicitly (e.g. the iteration-level vs. session-level split in [concepts/feedback-signals](../concepts/feedback-signals.md)). The **consolidation path** (files → harness → weights) is also a clean external framing of the wiki's [modular-decomposition](../overview.md) and [knowledge-accumulation](../concepts/knowledge-accumulation.md) themes.

## Connections

- [overview.md](../overview.md) — the wiki-wide synthesis this taxonomy complements (adds the persistence-horizon axis)
- [concepts/harness-optimization](../concepts/harness-optimization.md) — Tu's "Agent Harness" substrate, the wiki's densest region
- [concepts/knowledge-accumulation](../concepts/knowledge-accumulation.md) — the "External Files" substrate and the consolidation-upward dynamic
- [concepts/self-improvement-loop](../concepts/self-improvement-loop.md) — Tu's "Experience → State Change → Future Behavior" is the measure→…→repeat loop, indexed by *where the change lands*
- [sources/autogenesis](autogenesis.md) — a concrete protocol for the hard "Across Users × Harness" cell (portable, auditable self-modification)
- [sources/optimize-anything-omni](optimize-anything-omni.md), [sources/meta-harness](meta-harness.md) — canonical occupants of the Across-Sessions × Harness cell

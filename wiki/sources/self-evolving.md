---
title: "The What & When of Self-Evolving Agents (Xinming Tu)"
type: source
tags: [taxonomy, survey, framework, what-evolves, when-persists, consolidation, meta-framework]
sources: [self-evolving]
url: https://xinmingtu.github.io/blog/2026/self-evolving-agents/
author: Xinming Tu
published: 2026-07-22
last_updated: 2026-08-01
---

# The What & When of Self-Evolving Agents

**Blog**: [The What & When of Self-Evolving Agents](https://xinmingtu.github.io/blog/2026/self-evolving-agents/) — **Xinming Tu**, 2026-07-22 (host redirects to `xinmingtu.cn`).

Not a system and not a survey of results — a **taxonomy / analytical framework** for organizing how agents learn from experience. It maps almost perfectly onto this wiki's own axes, so it doubles as an external cross-cut of the whole collection (compare [Weng's harness-engineering survey](weng-harness-blog.md), which cross-cuts the same territory along a different ladder).

## The core framework: a 3 × 3 matrix

Two independent dimensions:

**What evolves** (the substrate that changes):
1. **External Files** — editable artifacts: notes, scripts, skills
2. **Agent Harness** — prompts, tools, workflows, control flow
3. **Model Weights** — parametric knowledge in checkpoints

**When updates persist** (the durability horizon; Tu also gives each an agent-centric name):
1. **Single Session** = *Intra-Task* — within one active trajectory (execution feedback)
2. **Across Sessions** = *Inter-Task* — longitudinal adaptation for one user/project (domain structure)
3. **Across Users** = *Inter-Agent* — population-level improvement benefiting everyone (peer discovery)

The core claim, distilled: learning **"must land somewhere [what], and it must last for some horizon [when]."** The elementary loop is `Experience → State Change → Future Behavior`.

## The nine cells, populated with wiki systems

Tu names exemplars in each cell; **bold-linked** ones have their own wiki page, the rest are external references he cites.

| What ↓ / When → | Single Session (Intra-Task) | Across Sessions (Inter-Task) | Across Users (Inter-Agent) |
|---|---|---|---|
| **External Files** | **[autoresearch](autoresearch-vs-hpo.md)** (edits training code mid-run), **[AlphaEvolve](alphaevolve.md)** (evaluator-guided program/proof improvement), MemGPT | Voyager (executable skill library), **[ACE](ace.md)** (itemized playbook), Mem0 | EinsteinArena (verified discoveries as shared seeds), Agent Skills (shared commons) |
| **Agent Harness** | Claude Code dynamic workflows, Recursive Language Models (the [RLM](halo.md) substrate) | **[Meta-Harness](meta-harness.md)**, DSPy, MIPRO, AgentOptimizer, Promptbreeder, **[GEPA](optimize-anything.md)**, **[AFlow](aflow.md)**, **[Self-Harness](self-harness.md)** | Claude Code / Codex defaults, **[ADAS](adas.md)**, **[Darwin Gödel Machine](dgm.md)**, **[Hyperagents](hyperagents.md)** |
| **Model Weights** | TTT-Discover (train during inference), ThetaEvolve (test-time RL on the search policy) | OpenClaw-RL (conversational feedback → RL) | Cursor Tab (accept/reject → RL), Autodata (agentic inference → training data), Self-play |

## Two ideas worth importing into the wiki

**1. The capability-consolidation path.** Discoveries can *migrate upward* through substrates as they prove durable: **task-local files → reusable harness logic → internalized weights**. Each step up trades reversibility/portability for durability/generalization (and higher cost). This is a clean framing for several wiki threads:
- [SKILL-0](skill-rl-skill0.md) literally consolidates in-context skills into weights (curriculum RL, progressive skill withdrawal) — the files→harness→weights path made concrete.
- [TRACE](trace.md) consolidates diagnosed capability gaps into LoRA adapters — a partial consolidation (into modular weights, not the base).
- [CORAL](coral.md)'s Skills store distills successful attempts into reusable procedures — consolidation *within* the files/harness layer, across agents.

**2. The horizon axis is orthogonal to "what is optimized."** The wiki's [overview](../overview.md) organizes mostly by *what* is optimized; Tu's *when* axis is a second coordinate the wiki had only implicitly (per-query vs. per-deployment vs. long-horizon). It cleanly separates otherwise-similar systems: [AutoReason](autoreason.md) and [Squeeze-Evolve](squeeze-evolve.md) are Single-Session (intra-task) output improvers; [SkillOpt](skillopt.md) and [Meta-Harness](meta-harness.md) are Across-Sessions harness improvers; [ADAS](adas.md)/[DGM](dgm.md) and platform defaults are Across-Users.

## Recursive self-improvement, per Tu

Tu frames RSI not as a separate mechanism but as **"the self-evolving loop applied to AI development itself"** — agents generating synthetic pre-training data, building evaluations, or improving training infrastructure that feeds the next model generation. This is the same recursive framing as [Weng's survey](weng-harness-blog.md) and the [STOP](stop.md)→[DGM](dgm.md)→[Hyperagents](hyperagents.md) lineage, viewed from the "what & when" angle (RSI = harness/weights changes that persist Across Users, then loop).

## Design tensions Tu flags

- **Portability vs. generalization**
- **Reversibility vs. durability** (why the consolidation path is a ratchet, not a free lunch)
- **Automation vs. governance** — he notes checkpoint bootstrapping (weights across-users) is "only partially autonomous today"

The framework "shows possibility, not obligation": most cells are populated, but nothing forces a discovery to climb the consolidation ladder.

## Key terminology

- **Meta-programming** — compressing repeated execution patterns into reusable DAGs (cf. [AFlow](aflow.md))
- **Consolidation** — promoting discoveries upward through substrates
- **Test-time training (TTT)** / **fast weights** — transient weight updates during inference
- **Parametric personalization** — LoRA-style per-user modules
- **Harness flywheel** — population failures → platform defaults (the Across-Users harness cell)

## Positioning in this wiki

Use this page as an **orientation lens**, not a result. When filing a new source, its two-coordinate address — *(what substrate, what horizon)* — is a fast way to place it against neighbors and spot empty cells. The wiki is dense in the **Agent-Harness × Across-Sessions** cell (most harness optimizers) and the **Files × Across-Sessions** cell (skills/playbooks/memory); it is thinner on **Model-Weights** cells (mostly [AgentFlow](agentflow.md), [TRACE](trace.md), [SKILL-RL/SKILL-0](skill-rl-skill0.md)), consistent with its focus on non-weight self-improvement.

## Connections

- [sources/weng-harness-blog](weng-harness-blog.md) — the other external map; Weng's *ladder* (instruction→context→workflow→harness-code→optimizer-code) vs. Tu's *2-D matrix* (what × when)
- [sources/skill-rl-skill0](skill-rl-skill0.md), [sources/trace](trace.md) — the consolidation path (files/harness → weights) made concrete
- [sources/adas](adas.md), [sources/dgm](dgm.md), [sources/hyperagents](hyperagents.md) — the Across-Users harness cell (self-modifying code)
- [sources/ace](ace.md), [sources/mce](mce.md) — the Files/Harness × Across-Sessions cells (context engineering)
- [concepts/knowledge-accumulation](../concepts/knowledge-accumulation.md) — the consolidation path is an accumulation-durability argument
- [concepts/self-improvement-loop](../concepts/self-improvement-loop.md) — Tu's `Experience → State Change → Future Behavior` is the same core loop, indexed by (what, when)
- [overview](../overview.md) — the wiki's own "what can be optimized" table is the *what* axis; Tu adds the *when* axis

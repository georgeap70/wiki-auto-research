---
title: "WebXSkill: Skill Learning for Autonomous Web Agents"
type: source
tags: [web-agents, skills, parameterized-programs, grounded-execution, guided-instructions, aiming-lab, WebArena, WebVoyager]
sources: [webxskill]
url: https://arxiv.org/abs/2604.13318
code: https://github.com/aiming-lab/WebXSkill
authors: Zhaoyang Wang, Qianhui Wu, Xuchao Zhang, Chaoyun Zhang, Wenlin Yao, Fazle Elahi Faisal, Baolin Peng, Si Qin, Suman Nath, Qingwei Lin, Chetan Bansal, Dongmei Zhang, Saravan Rajmohan, Jianfeng Gao, Huaxiu Yao
affiliations: Microsoft Research + UNC (aiming-lab)
last_updated: 2026-04-18
---

# WebXSkill: Skill Learning for Autonomous Web Agents

**arXiv:** [2604.13318](https://arxiv.org/abs/2604.13318) (April 2026)
**Code:** [github.com/aiming-lab/WebXSkill](https://github.com/aiming-lab/WebXSkill) — code placeholder, "coming soon"
**Labs:** Microsoft Research + UNC (aiming-lab, same lab as [[sources/agent0]])

## Summary

WebXSkill attacks the **grounding gap** in skill-based web agents: prior skill stores are either pure text (interpretable but not directly executable) or pure code (executable but opaque). WebXSkill introduces **executable skills** — parameterized action programs paired with step-level natural-language guidance — that can be used in two complementary modes.

The headline claim: on WebArena, skills deliver up to **+9.8 pts** over baseline; on WebVoyager, up to **+12.9 pts**.

## The Three-Stage Pipeline

### 1. Skill Extraction
Mine reusable action sequences from synthetic agent trajectories and abstract them into **parameterized skills**. Each skill captures a recurring web interaction pattern (e.g., search-and-filter, form-fill, navigate-and-extract) with its variables factored out.

### 2. Skill Organization
Skills are indexed in a **URL-based graph** — retrieval is context-aware: skills are surfaced based on which part of the web the agent is currently navigating, not on surface keyword match alone.

### 3. Skill Deployment
Two modes:

| Mode | What the agent receives | When useful |
|------|------------------------|-------------|
| **Grounded** | Parameterized executable program — multi-step actions run automatically | Deterministic, repeatable tasks |
| **Guided** | Step-by-step natural-language instructions derived from the skill | Novel contexts requiring flexibility; skills act as reference |

Dual deployment is the architectural novelty: the same skill artifact serves execution and instruction without duplication.

## Results

| Benchmark | Improvement over baseline |
|-----------|---------------------------|
| WebArena | up to **+9.8 pts** |
| WebVoyager | up to **+12.9 pts** |

## Key Distinctions vs. Other Sources

| Axis | WebXSkill | Comparable sources |
|------|-----------|-------------------|
| Skill representation | Parameterized program + NL guidance (dual) | [[sources/skill-rl-skill0]] SKILL-RL: NL + executable artifact; [[sources/coral]] Skills: NL description + executable |
| Deployment | Grounded (auto-exec) **or** guided (instructions) | Most other systems: one mode only |
| Retrieval | URL-graph context-aware | [[sources/skill-rl-skill0]] hierarchical (general → task-specific); [[sources/coral]] agent-directed read |
| Domain | Web agents specifically | Most skill systems target general agents |
| Skill acquisition | Mined from synthetic trajectories | [[sources/skill-rl-skill0]] distilled during RL training; [[sources/coral]] authored by peer agents |

## Relation to Self-Improvement

WebXSkill is closer to **skill-library construction** than to a fully closed self-improvement loop. The loop runs once (extract → organize → deploy), not iteratively. However, the extraction stage itself could be re-run as new trajectories accumulate, making this a single-step version of the continuous-skill-evolution pattern seen in [[sources/skill-rl-skill0]] (SkillBank) and [[sources/coral]] (Skills store).

Mapped onto measure → fail → propose → gate → repeat:

- **Measure** — agent executes tasks using current skill set
- **Fail** — implicit via trajectory outcomes during extraction
- **Propose** — skill extraction mines new patterns from trajectories
- **Gate** — skills are added if they abstract a recurring pattern; no explicit regression check
- **Repeat** — not formalized in the paper

## Relation to aiming-lab

WebXSkill shares the `aiming-lab` GitHub org with [[sources/agent0]]. Where Agent0 is two-agent co-evolution for general tool-integrated reasoning, WebXSkill is a skill-library approach for web agents specifically. Both emphasize bootstrapping capability without heavy supervised data.

## Open Questions

- The paper's code is placeholder as of April 2026 ("coming soon") — reproducibility is currently limited.
- The URL-graph retrieval is promising but has not been compared to pure embedding-based retrieval at scale.
- Can grounded and guided modes be combined (grounded where confident, guided where uncertain)?
- Does skill extraction from *synthetic* trajectories transfer to skills useful on real-world, noisy web pages?

## Connections

- [[concepts/knowledge-accumulation]] — skill library is a persistent knowledge store; dual representation is novel
- [[concepts/self-improvement-loop]] — single-pass skill construction; not a continuous loop but the building blocks for one
- [[sources/skill-rl-skill0]] — closest analog; WebXSkill's dual representation vs. SKILL-RL's hierarchical SkillBank vs. SKILL-0's weight internalization
- [[sources/coral]] — Skills store with NL + executable artifacts; similar idea, different domain and coordination model
- [[sources/agent0]] — same lab (aiming-lab); different mechanism (co-evolution vs. skill extraction)

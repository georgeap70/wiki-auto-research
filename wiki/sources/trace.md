---
title: "TRACE: Capability-Targeted Agentic Training"
type: source
tags: [capability-decomposition, LoRA, GRPO, synthetic-environments, routing, contrastive-trajectories, stanford-scaling-intelligence]
sources: [trace]
url: https://scalingintelligence.stanford.edu/blogs/trace/
paper: https://arxiv.org/abs/2604.05336
code: https://github.com/ScalingIntelligence/TRACE
authors: Hangoo Kang, Tarun Suresh, Jon Saad-Falcon, Azalia Mirhoseini
affiliations: Stanford Scaling Intelligence Lab
last_updated: 2026-04-18
---

# TRACE: Capability-Targeted Agentic Training

**Blog:** [scalingintelligence.stanford.edu/blogs/trace](https://scalingintelligence.stanford.edu/blogs/trace/)
**Paper:** [arXiv 2604.05336](https://arxiv.org/abs/2604.05336)
**Code:** [github.com/ScalingIntelligence/TRACE](https://github.com/ScalingIntelligence/TRACE)

## Core Thesis

LLM agents fail in complex environments not because they are generally weak, but because they are **missing specific capabilities**. Direct RL on the target environment (e.g., GRPO on τ²-Bench) provides only sparse outcome signals, which do not reveal *which* capabilities the agent lacks — so learning is sample-inefficient and often plateaus.

TRACE decomposes agent improvement into **capability-specific** training rather than domain-specific training.

## The Four-Stage Pipeline

```
(1) Analysis   → contrast successful vs. failed trajectories
                 diagnose specific capability gaps
(2) Generation → for each gap, synthesize an isolated training environment
                 that rewards exercising exactly that capability
(3) Optimization → train a lightweight LoRA adapter per capability via GRPO
                   in its synthetic environment
(4) Routing    → at inference, a classifier routes each task to the
                 appropriate adapter (base model used as classifier)
```

The diagnostic and generation steps are performed by supervising LLM agents (an "analysis agent" and a "generation agent") — not by the target agent itself diagnosing its own failures. This is a distinction from fully autonomous systems like [[sources/auto-harness]] or [[sources/coral]].

## Why Capability Decomposition Beats Direct GRPO

Direct GRPO on a complex target environment:
- The binary outcome reward is the only signal
- The reward does not decompose into per-capability credit
- Learning becomes a bottleneck: the agent improves slowly on every capability at once

TRACE:
- Each synthetic environment **isolates** one capability
- The reward in that environment directly correlates with exercising that capability
- Sample efficiency improves dramatically — per-capability signal is dense rather than sparse

## Results

| Benchmark | TRACE result | Baseline |
|-----------|--------------|----------|
| τ²-Bench | **+14.1 pts** over base agent; **+7.4 pts** over strongest baseline | — |
| τ²-Bench (sample efficiency) | **47.0%** pass rate at 5,120 rollouts | GRPO plateaus at **37.8%** |
| ToolSandBox | **+7 perfect scores** | Base +4 perfect scores |

Single adapters (before routing) already outperform direct GRPO — the decomposition itself delivers the gains, routing is the composition layer.

## Key Distinctions vs. Other Sources

| Axis | TRACE | Comparable sources |
|------|-------|-------------------|
| What is optimized | Weights — multiple **LoRA adapters**, one per capability | [[sources/agentflow]] trains a single planner; [[sources/skill-rl-skill0]] SKILL-0 internalizes skills into base weights |
| Feedback signal | Capability-isolated reward in synthesized environment (dense per capability, sparse globally) | [[sources/agentflow]] uses trajectory-level binary outcome broadcast; [[sources/meta-harness]] uses rich execution traces |
| Composition at inference | Router classifier over adapter set | [[sources/skill-rl-skill0]] SKILL-RL retrieves from external SkillBank; SKILL-0 eliminates retrieval |
| Autonomy | Supervised: analysis & generation agents diagnose and build environments | Most other systems: the agent drives its own improvement |

## Relationship to Skill-Based Approaches

TRACE's "capabilities" are closely related to [[sources/skill-rl-skill0|skills]] but differ in:

- **Granularity** — capabilities are broader behavioral competencies (e.g., "handle nested tool errors"), skills are narrower procedures
- **Representation** — capabilities are stored in LoRA adapter weights, skills in text (SkillBank) or base weights (SKILL-0)
- **Acquisition** — capabilities are discovered by *contrastive trajectory analysis*; skills are distilled from successful trajectories

Both represent a move toward **decomposing agent improvement into sub-problems with targeted training signals**.

## Relation to the Self-Improvement Loop

TRACE is best understood as a **semi-autonomous self-improvement system**:

- **Measure** — roll out base agent on target environment
- **Fail** — contrast failures with successes to diagnose capability gaps (done by LLM analysis agent)
- **Propose** — synthesize training environments targeting each gap (done by LLM generation agent)
- **Gate** — implicit: adapter is accepted if it improves performance in its synthetic environment
- **Compose** — router selects the appropriate adapter at inference

The agent being improved does not drive its own diagnosis — external LLM agents do. This makes TRACE less autonomous than [[sources/agent0]] or [[sources/coral]], but the capability-decomposition mechanism could in principle be integrated into a fully autonomous loop.

## Open Questions

- Does the adapter set scale — dozens of capabilities per environment is tractable, but what about hundreds?
- Router quality is load-bearing: can base-model classification keep up as the adapter set grows?
- Could the analysis agent itself be trained / improved via the same capability-decomposition approach (recursive TRACE)?
- Capabilities are identified *post-hoc* from observed failures; can they be predicted *a priori* from environment structure?

## Connections

- [[concepts/self-improvement-loop]] — semi-autonomous variant; external LLMs drive the diagnose/generate steps
- [[concepts/feedback-signals]] — capability-isolated rewards turn a globally sparse signal into locally dense signals; an alternative strategy to both rich traces and credit assignment
- [[concepts/knowledge-accumulation]] — capabilities persist as LoRA adapter weights; accumulation is in weight space with explicit modular structure
- [[sources/agentflow]] — both train policy weights; AgentFlow uses one planner with broadcast reward; TRACE uses per-capability adapters with isolated rewards
- [[sources/skill-rl-skill0]] — conceptually adjacent; SKILL-RL externalizes skills, SKILL-0 internalizes them into one model, TRACE modularizes them across adapters

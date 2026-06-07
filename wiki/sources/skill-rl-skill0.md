---
title: SKILL-RL and SKILL-0
type: source
tags: [skills, reinforcement-learning, curriculum-learning, knowledge-accumulation, ALFWorld, WebShop, skill-internalization]
sources: [skill0]
last_updated: 2026-04-08
---

# SKILL-RL and SKILL-0

Two companion papers addressing the same problem from different angles: **LLM-based agents fail to accumulate and reuse experience across episodes.**

---

## SKILL-RL

**Paper:** "SkillRL: Evolving Agents via Recursive Skill-Augmented Reinforcement Learning"
**arXiv:** [2602.08234](https://arxiv.org/abs/2602.08234)

### Core Idea

Rather than storing raw trajectories (expensive, noisy), SKILL-RL extracts **reusable behavioral patterns** — skills — from experience and builds a hierarchical SkillBank that co-evolves with the agent's policy during RL training.

### Architecture

**Three-layer system:**

1. **Experience-based distillation**: Instead of replaying raw rollouts, the system distills successful trajectories into compact skills with a natural-language description + executable artifact.

2. **Hierarchical SkillBank**: Skills are organized hierarchically — general heuristics at the top, task-specific procedures below. An adaptive retrieval strategy selects general skills by default and task-specific ones when relevant context is detected.

3. **Recursive evolution**: The SkillBank is not frozen after construction. It **co-evolves with the RL policy** — as the policy improves and discovers better strategies, the SkillBank is updated to reflect the current best practices. This is a recursive loop: better skills → better policy → better skills.

### Results

| Benchmark | Improvement over baseline RL |
|-----------|------------------------------|
| ALFWorld | part of +15.3% overall |
| WebShop | part of +15.3% overall |
| 7 search-augmented tasks | +15.3% avg over strong baselines |

- Robustness increases with task complexity (unlike baselines that degrade)
- Significantly reduces token footprint vs. raw-trajectory augmentation

---

## SKILL-0

**Paper:** "SKILL0: In-Context Agentic Reinforcement Learning for Skill Internalization"
**arXiv:** [2604.02268](https://arxiv.org/abs/2604.02268)
**Institution:** Zhejiang University

### Core Idea

SKILL-RL still requires **retrieval at inference time** — skills are stored externally and injected into context. SKILL-0 goes further: it **internalizes skills into model weights** so that at deployment there is no retrieval, no extra tokens, no external database.

### Architecture

**Dynamic curriculum learning:**

1. **Start**: Agent receives full skill context (all relevant skills in-context), plus interaction history rendered as visual context.
2. **Progressive withdrawal**: The curriculum linearly decays the skill budget — each training step, less skill context is provided.
3. **End state**: Agent operates with zero skill context — fully zero-shot.

The training signal is on-policy usefulness: skills that the agent actually uses to produce correct trajectories are reinforced; skills never used are pruned from the curriculum.

```
Training phase:
  step 0: full skill context → on-policy rollouts → RL update
  step N: partial skill context → on-policy rollouts → RL update
  step T: zero skill context → on-policy rollouts → RL update
```

### Results

| Benchmark | Improvement over baseline RL |
|-----------|------------------------------|
| ALFWorld | +9.7% |
| Search-QA | +6.6% |
| Token overhead | < 0.5k tokens/step |

- Eliminates inference-time retrieval entirely
- Smaller token footprint than SKILL-RL (no external skill injection)

---

## Comparison

| Dimension | SKILL-RL | SKILL-0 |
|-----------|----------|---------|
| Skill storage | External SkillBank | Model weights |
| Inference overhead | Retrieval + injection | None |
| Training mechanism | RL + recursive skill evolution | Curriculum RL with progressive withdrawal |
| Flexibility | SkillBank can be updated post-training | Requires retraining to update skills |
| Primary contribution | +15.3% across diverse tasks | Internalization: zero-shot deployment |

## Relationship to Knowledge Accumulation

Both papers attack the same failure mode: agents that reset to the same starting point every episode. But they propose complementary solutions:

- SKILL-RL: **external persistence** via SkillBank (same paradigm as [sources/coral](coral.md)'s Skills store, [sources/auto-harness](auto-harness.md)'s learnings.md)
- SKILL-0: **internal persistence** via weight update — the first system in this wiki where skills are stored in the model itself rather than a retrieval system

SKILL-0's internalization is conceptually adjacent to [sources/agentflow](agentflow.md)'s Flow-GRPO, but applied to **skill knowledge** rather than task policy.

## Connections

- [concepts/knowledge-accumulation](../concepts/knowledge-accumulation.md) — SkillBank (SKILL-RL) and weight internalization (SKILL-0) are two distinct persistence mechanisms; SKILL-0 is the only system in the wiki that stores accumulated knowledge in weights
- [concepts/feedback-signals](../concepts/feedback-signals.md) — RL reward signal is the primary feedback; both systems use sparse binary reward augmented by skill context
- [sources/coral](coral.md) — CORAL's Skills store is the closest analog to SKILL-RL's SkillBank; both extract reusable procedures from agent experience
- [sources/agentflow](agentflow.md) — both SKILL-0 and AgentFlow optimize model weights; AgentFlow targets task policy, SKILL-0 targets skill internalization

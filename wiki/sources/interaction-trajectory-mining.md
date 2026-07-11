---
title: "Automating SKILL.md Generation via Interaction Trajectory Mining"
type: source
tags: [skill-mining, computer-use, negative-result, knowledge-accumulation, transfer-failure, grpo]
sources: [interaction-trajectory-mining]
url: https://arxiv.org/abs/2606.20363
code: none
authors: Yuexing Hao, Xiaomin Li
arxiv: 2606.20363
last_updated: 2026-07-10
---

# Interaction Trajectory Mining for SKILL.md (arXiv 2606.20363)

## Summary

A **diagnostic / negative-result study** — rare and valuable in this wiki. It asks whether a reusable skill library (`SKILL.md`) for computer-using / GUI agents can be *mined automatically* from recorded interaction trajectories, and whether that mined library actually improves a downstream policy. The honest answer: **the structure is inspectable, but the transfer doesn't hold up.** It maps out precisely which components of the mining pipeline are too weak to deliver cross-domain gains.

This is the counterweight to the wiki's skill-centric optimism ([SkillOpt](skillopt.md), [WebXSkill](webxskill.md), [SKILL-RL](skill-rl-skill0.md), [CORAL](coral.md)'s Skills store): those systems *hand-author or co-evolve* skills against live feedback; this paper tries to mine them *offline* from logged trajectories, and finds that's much harder.

## Method: Three-Stage Mining Pipeline

1. **Segmentation** — cut GUI interaction trajectories into candidate sub-tasks (a learned boundary detector).
2. **Clustering** — group segments into candidate reusable skills.
3. **Policy training** — train a downstream policy with **GRPO** on skill-aware annotations, then measure whether the mined skills help.

## What Is Optimized / Feedback

| Axis | This work |
|------|-----------|
| Optimization target | A mined `SKILL.md` library + a policy trained on it |
| Feedback signal | Offline reward model over logged trajectories (no live rollouts) |
| Loop structure | One-shot offline pipeline (segment → cluster → train), not an iterative loop |
| Human involvement | Evaluation/diagnosis only |

## Results (mixed — reported honestly)

- **Readability holds**: 5 of 8 clusters reach ≥0.95 purity against InteraSkill Workflow labels — the mined structure is human-inspectable.
- **Transfer fails**: GRPO on skill annotations moved skill-step accuracy only 18.5% → 20.5% on InteraSkill, showed minimal change on BrowseComp+, and **underperformed a simple frequency-prior baseline**.
- **Diagnosis**: the boundary detector, the segment representation, and the offline reward model are each too weak for reliable cross-domain improvement.

## Why It Matters

- **A cautionary bound on automatic skill accumulation.** It sharpens where skill-based [knowledge accumulation](../concepts/knowledge-accumulation.md) is fragile: the bottleneck is not *representing* skills (clusters are legible) but *segmenting* trajectories and *scoring* skills offline. Systems that succeed do so with live, dense feedback ([SkillOpt](skillopt.md)'s held-out gate, [CORAL](coral.md)'s grader) — not offline mining.
- **Offline mining vs. live optimization.** It implicitly argues that the [feedback signal](../concepts/feedback-signals.md) matters more than the artifact: an offline reward model over logged traces is too lossy to drive transfer, echoing the field's rich-live-feedback thesis from the failure side.
- Fits the failure-mode literature that [Lilian Weng's survey](weng-harness-blog.md) foregrounds (weak evaluators, offline reward-model limits).

## Connections

- [concepts/knowledge-accumulation](../concepts/knowledge-accumulation.md) — negative bound: automatic offline mining of a skill store does not (yet) transfer
- [concepts/feedback-signals](../concepts/feedback-signals.md) — offline reward-model feedback is too weak; supports the live-rich-feedback thesis from the failure side
- [sources/skillopt](skillopt.md), [sources/webxskill](webxskill.md), [sources/skill-rl-skill0](skill-rl-skill0.md) — the skill-centric systems this study cautions against over-generalizing
- [sources/weng-harness-blog](weng-harness-blog.md) — the "weak evaluators" and negative-results themes it exemplifies

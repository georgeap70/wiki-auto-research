---
title: AgentFlow
type: source
tags: [on-policy-rl, modular-agent, credit-assignment, training, flow-grpo, multi-turn]
sources: [agentflow]
last_updated: 2026-04-07
---

# AgentFlow

**Paper:** arXiv 2510.05592 — ICLR 2026 oral
**Project:** https://agentflow.stanford.edu/
**Orgs:** Stanford University, Texas A&M University, UC San Diego, Lambda Labs
**Authors:** Zhuofeng Li, Haoxiang Zhang, Pan Lu, Yejin Choi, James Zou (among others)

## What It Is

AgentFlow is a trainable agentic framework that optimizes agent behavior **during live multi-turn interaction** via on-policy RL. Unlike every other system in this wiki — which optimize harnesses, prompts, or architecture code offline — AgentFlow trains the agent's **policy weights** in-the-flow of actual task execution.

The framework decomposes the agent into four specialized modules:

| Module | Role |
|--------|------|
| **Planner** | Generates sub-goals and selects tools; the module that is trained |
| **Executor** | Invokes the selected tools |
| **Verifier** | Checks whether the current result satisfies the goal |
| **Generator** | Produces the final solution |

These modules communicate through a shared evolving memory that accumulates context across turns.

## Training Method: Flow-GRPO

The key algorithmic contribution is **Flow-GRPO** (Flow-based Group Refined Policy Optimization). The core challenge: multi-turn agent interactions have a credit assignment problem — early decisions contribute to late outcomes, but naive RL would have no way to attribute a terminal reward back to intermediate steps.

Flow-GRPO's solution:
1. **Trajectory-level outcome signal**: binary reward (correct / incorrect final answer) evaluated at the end of a full trajectory
2. **Broadcast to every step**: this outcome signal is assigned to every decision point along the trajectory, not just the terminal step
3. **Group-normalized advantages**: across a group of sampled trajectories, normalize advantage estimates to reduce variance
4. **Token-level importance ratios**: convert multi-turn optimization into tractable single-turn policy gradient updates

This allows on-policy training directly within live environments — unlike offline RL methods that train on pre-collected trajectories. The planner continuously improves while operating.

## Feedback Signal

Formally a **scalar binary reward** — but with a critical distinction: the trajectory-level outcome is **credit-assigned to every intermediate action**. This is not the same as a dense reward; it's a sparse scalar redistributed across the sequence.

This is a notable contrast to the rest of the wiki's rich-feedback thesis (see [[concepts/feedback-signals]]): AgentFlow achieves strong results with only a terminal binary signal, suggesting that **credit assignment mechanism** matters as much as signal richness.

## Loop Structure

The loop is **in-the-flow** rather than the standard batch measure→fail→propose→gate cycle:

```
(live execution) planner → executor → verifier → generator
                    ↑                                   |
                    └── Flow-GRPO policy update ← reward
```

There is no separate "improvement phase" outside of task execution. The policy improves *while* solving tasks, not between task batches.

## Benchmarks and Results

Backbone model: 7B parameters. Evaluated across four categories:

| Category | Benchmarks | Accuracy gain |
|----------|-----------|---------------|
| Knowledge-intensive search | Bamboogle, HotpotQA, Musique | +14.9% |
| Agentic reasoning | GAIA | +14.0% |
| Mathematical logic | AIME, AMC, Game of 24 | +14.5% |
| Scientific reasoning | GPQA, MedQA | +4.1% |

Performance sometimes surpasses GPT-4o on these benchmarks despite a 7B backbone.

## Key Distinctions vs. Other Sources

| Axis | AgentFlow | Most other sources |
|------|-----------|-------------------|
| What is optimized | Policy weights (on-policy RL) | Harness, prompts, code, architecture |
| Feedback signal | Binary scalar (broadcast) | Rich traces, textual gradients, or textual loss |
| Loop timing | In-the-flow (during execution) | Between-task batch cycle |
| Human involvement | Fully autonomous | Fully autonomous |
| Gating | Implicit (RL convergence) | Explicit threshold / Pareto / held-out eval |

## Connections

- [[concepts/self-improvement-loop]] — introduces a qualitatively new loop type: in-the-flow on-policy training
- [[concepts/feedback-signals]] — challenges the rich-feedback-wins thesis with a scalar-but-credit-assigned approach
- [[sources/asi-evolve]] — both optimize more fundamental parameters (weights vs. architecture/RL algorithm); both are fully autonomous
- [[sources/agent0]] — both use multi-turn loops, but Agent0 bootstraps from zero data via two-agent co-evolution while AgentFlow trains a fixed modular architecture

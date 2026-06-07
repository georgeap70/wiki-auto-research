---
title: "Agent0: Unleashing Self-Evolving Agents from Zero Data via Tool-Integrated Reasoning"
type: source
tags: [self-improvement, two-agent, co-evolution, tool-use, zero-data]
sources: [agent0]
url: https://arxiv.org/abs/2511.16043
code: https://github.com/aiming-lab/Agent0
authors: Peng Xia, Kaide Zeng, Jiaqi Liu, Can Qin, Fang Wu, Yiyang Zhou, Caiming Xiong, Huaxiu Yao
affiliations: Salesforce, UNC Chapel Hill
last_updated: 2026-04-04
---

# Agent0: Self-Evolving Agents from Zero Data

## Summary

Agent0 demonstrates that agents can bootstrap complex reasoning and tool-use capabilities from **zero human-curated data**, using only a two-agent co-evolution loop.

## Key Idea

Two agents are placed in competition:

1. **Curriculum agent** — generates progressively harder tasks
2. **Executor agent** — solves them using tool-integrated reasoning

When the executor improves at tool use, the curriculum agent is forced to escalate task difficulty. This creates a self-reinforcing feedback loop: neither agent needs human-designed training data.

External tools are integrated throughout — they amplify the executor's capabilities, which in turn raises the bar the curriculum agent must meet.

## Method

- Start from zero labeled data
- Curriculum agent samples tasks calibrated to the executor's current capability level
- Executor attempts tasks using available tools
- Performance signals drive mutual escalation
- Loop continues until convergence or target capability level

## Key Insight

The co-evolution structure eliminates the bottleneck of **human data curation**. The agents are each other's training environment. This is qualitatively different from self-play (same agent, same game) — the curriculum agent's goal is specifically to stretch the executor, not to win.

## Relation to Self-Improvement

Agent0 represents **[self-improvement](../concepts/self-improvement-loop.md) at the task-solution level** via two-agent co-evolution. The loop:
- `measure` — executor performance on curriculum tasks
- `fail` — tasks where executor fails
- `propose` — curriculum agent generates new harder tasks; executor updates via tool-integrated reasoning
- `gate` — implicit: escalation only happens when executor has mastered current level

## Connections

- entities/optimize-anything also starts from minimal initialization and expands via LLM proposals
- entities/evox uses population-based evolution rather than two-agent dynamics
- [concepts/feedback-signals](../concepts/feedback-signals.md) — Agent0 uses implicit feedback (pass/fail on tasks) rather than rich diagnostics

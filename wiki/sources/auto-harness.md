---
title: "auto-harness — NeoSigma AI (Repo + Blog)"
type: source
tags: [self-improvement, harness, agentic-loop, regression-gating, practical]
sources: [auto-harness, self-improving]
urls:
  - https://github.com/neosigmaai/auto-harness
  - https://www.neosigma.ai/blog/self-improving-agentic-systems
authors: Gauri Gupta, Ritvik Kapila
org: NeoSigma AI
last_updated: 2026-04-04
---

# auto-harness (NeoSigma AI)

## Summary

`auto-harness` is an open-source framework that lets a coding agent **improve itself overnight** without human code edits. Combined with the NeoSigma AI blog post, it provides both the implementation and the conceptual framing for self-improving agentic systems in production.

## The System

Given:
- A benchmark (e.g., [[entities/tau3|Tau3]])
- The agent's own `agent/agent.py`

The agent executes a loop:
1. Run tasks on the benchmark
2. Analyze failures
3. Modify its own **system prompt** and **tools**
4. Gate changes through a regression suite (≥80% on prior tasks)
5. Store learnings in `workspace/learnings.md`
6. Repeat

Everything runs inside **Docker** for isolation. The agent never touches its own core model weights — all improvement is at the harness/scaffolding layer.

## Results

| Benchmark | Baseline | Final | Gain | Batches | Experiments |
|-----------|---------|-------|------|---------|-------------|
| Tau3 | 0.560 | 0.780 | +39.3% | 18 | 96 |

Underlying model (GPT-5.4) was **held constant** throughout. All gains came from harness iteration alone.

## Conceptual Framing (Blog Post)

The blog argues that the **bottleneck in AI engineering has shifted**: it's no longer code generation, it's validation and maintenance. Self-improvement infrastructure closes the loop automatically.

The four-phase cycle described:
1. **Mine** — extract failures from production execution traces
2. **Cluster** — group by root cause
3. **Convert** — turn clusters into eval test cases
4. **Optimize** — run improvement loop with regression gating

## Key Design Choices

- **[[concepts/regression-gating|Regression gating at 80%]]**: changes are rejected if they degrade previously passing tasks
- **`learnings.md`**: persistent memory that survives context resets across optimization runs
- **No model retraining**: improvement is entirely at the scaffolding layer — makes the approach model-agnostic
- **Docker isolation**: prevents the agent from accidentally breaking its own environment

## Connections

- [[entities/meta-harness]] optimizes the same layer (system prompts + scaffolding) but with richer diagnostic context
- [[sources/autoharness-arxiv]] (different paper, similar name) focuses on synthesizing constraint code rather than iterating on the whole harness
- [[concepts/harness-optimization]] — this is a primary example
- [[concepts/feedback-signals]] — uses production traces for failure mining (rich signal), but scalar pass/fail for gating

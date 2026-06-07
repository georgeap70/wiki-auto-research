---
title: HALO — Hierarchical Agent Loop Optimization
type: source
tags: [harness-optimization, traces, opentelemetry, rlm, production, context-labs, appworld]
sources: [halo]
last_updated: 2026-04-25
---

# HALO

**Repo**: [github.com/context-labs/halo](https://github.com/context-labs/halo)
**Author**: Context Labs (MIT license)

## One-line summary

Production-trace-driven harness optimization: collect OpenTelemetry traces from a deployed agent, feed them through a Recursive Language Model (RLM) tuned for trace analysis, generate diagnostic reports, hand the reports to a coding agent that edits the harness, redeploy.

## Core thesis

General-purpose coding agents (Claude Code, etc.) struggle to optimize harnesses from execution traces because:
- Traces are extremely long
- Trace analysis requires specialized pattern recognition across many runs
- Generic agents tend to overfit to isolated errors rather than identify systemic patterns

HALO's answer: a **dedicated trace-analysis component** (the HALO-RLM engine) sits between raw traces and the harness-editing agent. The RLM is specialized for systemic-pattern analysis — its job is *only* to surface failure modes that occur across executions, not to fix them.

## Loop architecture

```
1. Trace Collection      — agent runs in production, OpenTelemetry traces collected
2. Engine Analysis       — HALO-RLM engine ingests traces
3. Problem Decomposition — engine identifies common failure modes across executions
4. Code Generation       — diagnostic report → coding agent → harness modifications
5. Redeployment          — updated harness deployed; new traces flow back to step 1
```

This is a clean separation of concerns: the RLM understands *what's broken at scale*; the coding agent understands *how to change code*. Neither tries to do both.

## What is being optimized

The **harness** — surrounding infrastructure, prompts, tool wiring — not model weights. This is the same target as [sources/auto-harness](auto-harness.md) and [sources/meta-harness](meta-harness.md), but with explicit production-trace input as the primary feedback channel.

## Failure modes the RLM surfaces

- Hallucinated tool calls
- Redundant tool arguments
- Refusal loops
- Semantic correctness issues

These are *systemic* patterns visible only across many runs, not single-trace bugs.

## Key results — AppWorld benchmark

AppWorld is a multi-app service benchmark with separate dev and test splits.

| Model | Dev success (before → after) | Test (delta) |
|-------|------------------------------|--------------|
| Gemini 3 Flash | 36.8% → 52.6% (+15.8pp) | +10.7pp |
| Sonnet 4.6 | 73.7% → 89.5% (+15.8pp) | +10.7pp |

**Test gains are smaller but substantial** — the harness changes generalize, so optimization isn't dataset overfit. Notably, the same +15.8pp dev gain appears for both a frontier and a smaller model, suggesting the harness was the bottleneck for both.

## Connections

- [sources/auto-harness](auto-harness.md) — closest analog: NeoSigma also uses production traces to drive overnight harness rewrites; HALO formalizes this as a methodology with a specialized trace-analysis component (RLM) and a tracing standard (OpenTelemetry)
- [sources/meta-harness](meta-harness.md) — both rely on rich trace context for proposal quality; HALO emphasizes *cross-trace* pattern analysis vs. Meta-Harness's deep single-trace inspection
- [sources/trace](trace.md) — TRACE also contrasts trajectories to find capability gaps, but trains LoRA adapters; HALO edits harness code instead. Different proposal target, similar diagnosis primitive (cross-run pattern extraction)
- [concepts/harness-optimization](../concepts/harness-optimization.md) — production-trace-driven variant
- [concepts/feedback-signals](../concepts/feedback-signals.md) — OpenTelemetry traces are among the richest feedback signals in the wiki; the RLM is itself a *signal-compressor* (turning many traces into a finite-length report)
- [concepts/self-improvement-loop](../concepts/self-improvement-loop.md) — explicit deploy → trace → analyze → edit → redeploy production loop; one of the few systems explicitly designed around live deployment traffic

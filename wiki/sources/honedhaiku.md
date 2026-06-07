---
title: HonedHaiku — Prompt Optimization for Bug Fixing
type: source
tags: [prompt-optimization, gepa, harness-optimization, coding, benchmark, goldilocks]
sources: [honedhaiku]
last_updated: 2026-04-22
---

# HonedHaiku

**Post**: [tim.waldin.net/blog/2026-04-19-hone-haiku-20pp](https://tim.waldin.net/blog/2026-04-19-hone-haiku-20pp)  
**Author**: Tim Waldin

## One-line summary

GEPA-based system prompt optimization raises Claude Haiku's bug-fixing solve rate from 65% to 85% on unseen bugs — +19.7 percentage points — using only prose-level prompt changes, in 4 iterations.

## System design

Three components orchestrated by "Hone" (~300 lines of glue):

| Component | Role |
|-----------|------|
| GEPA | Proposes mutated prompt candidates via Pareto-style mutation and selection |
| Harness | Wrapper API that runs any of 6 AI coding tools (Claude Code, Codex, OpenCode, Gemini, Cursor, Aider) with the candidate prompt |
| Agentelo | Challenge runner; scores fixes as `tests_fixed / tests_broken_before` against real PR test suites |

Loop: GEPA proposes → Harness executes → Agentelo scores → highest variant becomes next parent.

## What is being optimized

The **system prompt only** — no weights, no tools, no scaffolding code. Seed was 14 words. Final prompt expanded to a structured 6-step methodology.

## Feedback signal

Win rate on real PR test suites: whether the agent's fix passes all originally-failing tests without breaking any passing ones. This is a pass/fail binary per test, aggregated as a ratio.

## Results

| Set | Seed | Optimized | Delta |
|-----|------|-----------|-------|
| Training (20 challenges, 5 repos) | 54.76% | 91.76% | +36.99pp |
| Holdout (9 unseen bugs, 3 samples each) | 64.96% | 84.62% | **+19.66pp** |

Converged in 4 iterations (out of 20 allocated). No holdout regressions across all three samples.

## Key findings

### GEPA's diagnosis
The optimizer identified a single dominant failure mode: *"Haiku fixes the first visible test and declares victory."* The final prompt addresses this with explicit instructions to enumerate *all* failing tests, trace *each* to root cause, check for multiple bug sites, and iterate until every test passes.

### The Goldilocks band
Prompt optimization has a performance ceiling and floor:

| Baseline regime | Optimization outcome |
|-----------------|---------------------|
| < ~50% | Model can't execute complex methodologies; no gain |
| 50–70% | Productive zone: prompts can close the gap |
| > ~85% | Saturation; prompt is no longer the bottleneck |

Claude Haiku's 65% baseline placed it squarely in the productive band. Stronger models (already at 85%+) would show minimal gains.

### Training diversity is critical
A 3-challenge training run overfitted and regressed on holdout. Scaling to 20 challenges across 5 repositories fixed generalization. This is the same insight as GEPA in [sources/optimize-anything](optimize-anything.md): diverse training set prevents the optimizer from learning dataset-specific heuristics.

## Connections

- [sources/optimize-anything](optimize-anything.md) — GEPA is the optimization primitive used here; same Pareto-mutation-selection loop applied to prompts rather than code
- [sources/deep-research](deep-research.md) — also applies GEPA to system prompt optimization; different domain (research pipeline vs. coding)
- [concepts/harness-optimization](../concepts/harness-optimization.md) — system prompt optimization is the simplest form of harness optimization
- [concepts/evolutionary-optimization](../concepts/evolutionary-optimization.md) — GEPA is the evolutionary algorithm underlying both this work and optimize_anything
- [concepts/feedback-signals](../concepts/feedback-signals.md) — PR test suite pass/fail is a rich, grounded signal (real code, real tests)

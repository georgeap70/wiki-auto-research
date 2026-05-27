---
title: "Meta-Harness: End-to-End Optimization of Model Harnesses (Stanford IRIS)"
type: source
tags: [harness-optimization, rich-feedback, system-prompt, scaffolding, stanford]
sources: [meta-harness]
url: https://yoonholee.com/meta-harness/
code: https://github.com/stanford-iris-lab/meta-harness-tbench2-artifact
authors: Yoonho Lee, Roshen Nair, Qizheng Zhang, Kangwook Lee, Omar Khattab, Chelsea Finn
org: Stanford IRIS Lab
last_updated: 2026-04-04
---

# Meta-Harness (Stanford IRIS Lab)

## Summary

Meta-Harness automates the improvement of the **configuration layer around LLMs** — system prompts, interaction patterns, and tool scaffolding — using Claude Code as an optimizer agent. Its central argument: **rich diagnostic access** (not just reward signals) is the key enabler for effective autonomous optimization.

## What Is a Harness?

A "harness" is everything that wraps a base LLM: system prompts, tool definitions, interaction protocols, output parsers. Harnesses are typically written by humans and rarely updated. Meta-Harness treats the harness as an artifact to be optimized.

## Method

The optimizer agent (Claude Code) is given:
- Full **source code** of the harness
- **Scores** from prior candidates
- Complete **execution traces** (up to 10M tokens of context)

It then:
1. Diagnoses specific failure modes from traces
2. Proposes targeted harness edits (not random perturbations)
3. Tests edits on held-out tasks
4. Iterates

The agent uses standard CLI tools — diff, grep, read/write — to inspect and modify the harness.

## The Key Design Choice: Rich Context

Prior approaches compressed diagnostic information into scalar scores or brief summaries before passing to the optimizer. Meta-Harness passes everything — full source, full traces, all prior scores — directly to the agent.

Why this matters:
- The optimizer can **trace a specific failure** back to a specific line in the harness
- Proposed edits are **surgical**, not exploratory
- The agent improves *faster per iteration* because it understands *why* failures occur

This is the empirical backing for the [[concepts/feedback-signals|rich feedback signals]] thesis.

## Relation to Other Systems

| System | What it optimizes | Feedback type |
|--------|-------------------|---------------|
| [[sources/auto-harness]] | System prompt + tools | Pass/fail + learnings.md |
| Meta-Harness | Full harness (prompt + scaffolding) | Rich traces + full source |
| [[sources/autoharness-arxiv]] | Constraint code | Environmental action feedback |

## Connections

- [[concepts/harness-optimization]] — central example of full-harness optimization
- [[concepts/feedback-signals]] — strongest argument in the literature for rich diagnostic traces over scalar reward
- [[sources/optimize-anything]] — also formalizes rich feedback as ASI; converging ideas from different groups
- [[entities/claude-code]] — used as the optimizer agent

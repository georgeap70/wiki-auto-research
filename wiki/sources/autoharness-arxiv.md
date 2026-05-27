---
title: "AutoHarness: Improving LLM Agents by Automatically Synthesizing a Code Harness"
type: source
tags: [harness, code-synthesis, constraint-enforcement, self-improvement, TextArena]
sources: [auto-harness]
url: https://arxiv.org/abs/2603.03329
code: none
authors: Xinghua Lou, Miguel Lázaro-Gredilla, Antoine Dedieu, Carter Wendelken, Wolfgang Lehrach, Kevin P. Murphy
org: Google DeepMind
last_updated: 2026-04-04
---

# AutoHarness (arXiv paper)

> **Note**: This is a different project from [[sources/auto-harness|NeoSigma's auto-harness repo]], despite the similar name.

## Summary

AutoHarness addresses a specific, concrete failure mode: **78% of Gemini-2.5-Flash losses** in a chess competition were caused by illegal moves. Instead of manually writing constraint-enforcement code, the model generates and refines its own "harness" — a protective code wrapper — using environmental feedback.

## The Problem

In constrained action environments (games, APIs, formal systems), LLM agents frequently produce **invalid actions**. Writing constraint-enforcement code by hand for 145+ different games is infeasible. The question: can an agent write its own constraint code?

## Method

1. Agent attempts tasks in a constrained environment (TextArena — 145 games)
2. Environment returns **feedback on invalid actions** (not just win/loss)
3. Agent iteratively generates, tests, and refines a "harness" — code that enforces valid actions before submission
4. The harness is persistent: once synthesized for a game, it carries over to future runs

## Key Results

- Smaller models equipped with auto-synthesized harnesses **outperform larger foundation models** without harnesses
- Demonstrates that **[[concepts/code-synthesis|self-generated code]] can substitute for expensive inference** at a smaller model scale
- The harness effectively externalizes constraint knowledge from the model's weights into code

## Key Insight

Constraint violations are a type of structured failure with **rich diagnostic signal** (which move was illegal, why). AutoHarness exploits this: environmental feedback is specific enough that the agent can diagnose the failure and write corrective code, rather than just adjusting its output distribution.

This is [[concepts/self-improvement-loop|self-improvement]] at the **code policy layer** — the agent improves not by changing its reasoning, but by building a more constrained action space for itself.

## Connections

- [[concepts/harness-optimization]] — primary example of constraint-harness synthesis
- [[concepts/feedback-signals]] — rich environmental feedback (illegal move signals) drives iterative refinement
- [[sources/auto-harness]] (NeoSigma repo) — optimizes the whole harness layer; this paper focuses narrowly on constraint-enforcement code
- [[entities/optimize-anything]] — similarly argues that rich "actionable side information" beats scalar reward

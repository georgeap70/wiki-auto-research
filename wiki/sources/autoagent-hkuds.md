---
title: "AutoAgent (HKUDS) — Natural Language Agent Builder"
type: source
tags: [agent-building, natural-language, multi-agent, workflow, gaia, self-developing, no-code]
sources: [autoagent]
urls:
  - https://github.com/hkuds/autoagent
org: HKUDS
last_updated: 2026-04-04
---

# AutoAgent (HKUDS)

## Summary

AutoAgent is an open-source framework that lets users **build and deploy LLM-based agent systems through natural language alone** — no manual coding or technical configuration required. Its relevance to self-improving systems lies in its "self-developing architecture": agents can iteratively refine themselves through feedback without the user writing any code.

## The System

Three modes of operation:

1. **User Mode (Deep Research)**: A pre-built multi-agent pipeline for information retrieval, analysis, and report generation. Positioned as a cost-effective alternative to subscription services like Claude 3.5 Pro.
2. **Agent Editor**: Define an individual agent in natural language; the system auto-profiles its requirements and generates tool definitions.
3. **Workflow Editor**: Compose multi-agent workflows conversationally — the system handles orchestration, routing, and inter-agent communication.

Infrastructure is containerized (Docker) with multi-provider model support via LiteLLM (Anthropic, OpenAI, Deepseek, Mistral, Gemini, others).

## Self-Developing Architecture

Agents can iteratively improve through feedback mechanisms, supporting both single-agent and collaborative multi-agent scenarios. The notable aspect for this wiki: **the entire harness (tools, prompts, orchestration) is generated and modified from natural language**, making the optimization interface purely linguistic rather than code-level. This is a different point on the human-involvement spectrum than other systems here — the agent still needs a human to direct changes via natural language, but the human never touches code.

## Results

| Benchmark | Result |
|-----------|--------|
| GAIA | Comparable to Claude 3.5 Sonnet on deep research tasks |
| MultiHopRAG (Agentic-RAG) | Evaluated; specific scores not published in README |

## Key Design Choices

- **Natural language as the complete optimization interface**: prompts, tools, and workflows are all specified and modified via dialogue
- **Docker isolation**: all agent runs containerized
- **LiteLLM multi-provider**: decouples the framework from any single model vendor
- **No-code agent construction**: lowers barrier to building and iterating on agent harnesses

## Relationship to Self-Improvement

AutoAgent sits at the boundary of this wiki's core topic. It does not autonomously self-optimize on a benchmark overnight (contrast: [sources/auto-harness](auto-harness.md), [sources/autoagent-kevinrgu](autoagent-kevinrgu.md)). Instead, it demonstrates that **natural language can be sufficient as the full interface for agent harness construction and iteration** — the human directs, the system implements. This is a human-in-the-loop point on the autonomy spectrum.

## Connections

- [concepts/harness-optimization](../concepts/harness-optimization.md) — the whole harness is NL-specified, a different approach to harness construction vs. autonomous code editing
- [sources/autoagent-kevinrgu](autoagent-kevinrgu.md) — same name, different project; kevinrgu/autoagent automates the optimization loop autonomously while this one requires NL direction
- [sources/meta-harness](meta-harness.md) — both treat the full harness as optimizable; Meta-Harness does it autonomously, this one via NL dialogue

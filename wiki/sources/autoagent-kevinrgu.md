---
title: "AutoAgent (KevinRGU) — Meta-Agent Harness Optimizer"
type: source
tags: [harness-optimization, meta-agent, hill-climbing, self-improvement, agentic-loop, score-driven]
sources: [autoagent2]
urls:
  - https://github.com/kevinrgu/autoagent/tree/main
last_updated: 2026-04-04
---

# AutoAgent (KevinRGU)

## Summary

A framework for **autonomous agent harness optimization via a meta-agent hill-climbing loop**. The key innovation is a clean abstraction boundary: human engineers write a high-level directive (`program.md`), and the meta-agent autonomously edits the implementation (`agent.py`) to maximize a benchmark score — without the human ever touching the code.

## The System

The loop:

1. **Human writes `program.md`**: a Markdown directive describing the desired agent behavior (the only human input after setup)
2. **Meta-agent reads the directive**, inspects the current `agent.py`, runs benchmarks, analyzes failures
3. **Meta-agent proposes edits** to the system prompt, tool definitions, routing logic, or configuration
4. **Score-driven gating**: keep changes that improve the numeric benchmark score, discard those that don't
5. **Repeat** until convergence or time limit

All agent runs are containerized (Docker). Results are tracked in `results.tsv` for experiment logging.

## Key Design Choice: Abstraction Boundary

The central novelty is the **separation of concerns** between:

| File | Who writes it | What it contains |
|------|--------------|-----------------|
| `program.md` | Human engineer | High-level behavioral directive |
| `agent.py` | Meta-agent (auto-edited) | Implementation: system prompt, tools, routing |

This treats agent engineering as a **searchable design space** rather than hand-crafted work. The human's role is reduced to specifying intent; the meta-agent handles implementation search. Contrast with [sources/auto-harness](auto-harness.md), where there is no explicit directive file — the agent infers what to improve from benchmark failures alone.

Additionally, `agent.py` uses **marked editable and fixed sections** to constrain the search space: the meta-agent can only modify designated regions, preventing accidental breakage of core infrastructure.

## Results

No published benchmark numbers in the repository. The framework is presented as infrastructure rather than a system with demonstrated empirical results. The approach is validated conceptually via architecture and an experiment logging visualization.

## Comparison to Related Systems

| System | Directive format | What is optimized | Benchmark results |
|--------|-----------------|-------------------|------------------|
| **AutoAgent (KevinRGU)** | `program.md` (NL Markdown) | Full harness via hill-climbing | None published |
| [sources/auto-harness](auto-harness.md) | None (implicit from failures) | System prompt + tools | +39% on Tau3 |
| [sources/meta-harness](meta-harness.md) | None (implicit from source + traces) | Full harness | Rich trace → better proposals |
| [sources/autoagent-hkuds](autoagent-hkuds.md) | Conversational NL (human-directed) | Full harness | GAIA comparable to Claude 3.5 |

## Connections

- [concepts/harness-optimization](../concepts/harness-optimization.md) — primary example of harness optimization via meta-agent loop
- [concepts/self-improvement-loop](../concepts/self-improvement-loop.md) — instantiates the measure→fail→propose→gate cycle at the harness layer
- [concepts/regression-gating](../concepts/regression-gating.md) — uses score-driven gating (numeric improvement required)
- [concepts/feedback-signals](../concepts/feedback-signals.md) — gating signal is scalar (benchmark score); failure analysis is implicit
- [sources/autoagent-hkuds](autoagent-hkuds.md) — same name, different project; HKUDS is NL-directed, this one is fully autonomous

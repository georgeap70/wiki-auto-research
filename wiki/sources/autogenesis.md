---
title: "Autogenesis: A Self-Evolving Agent Protocol"
type: source
tags: [protocol, self-evolving, lifecycle-management, lineage, rollback, MCP, A2A, versioning]
sources: [autogenesis]
url: https://arxiv.org/abs/2604.15034
code: none published
authors: Wentao Zhang
last_updated: 2026-04-18
---

# Autogenesis: A Self-Evolving Agent Protocol

**arXiv:** [2604.15034](https://arxiv.org/abs/2604.15034) (April 2026)

## Summary

Autogenesis introduces the **Autogenesis Protocol (AGP)** — a meta-protocol that treats the *entities* an agent uses (prompts, tools, memory, etc.) as first-class, versioned resources with explicit lifecycles, and adds an evolution layer with auditable lineage and rollback. Where MCP and A2A define how agents *use* tools or talk to other agents, AGP defines how agents *modify their own definitions* safely.

Think of it as "git + CI for agent internals."

## The Two-Layer Architecture

### RSPL — Resource Substrate Protocol Layer

Models five entity types as protocol-registered resources with explicit state, lifecycle, and versioned interfaces:

| Entity type | What it is |
|-------------|-----------|
| **Prompts** | System instructions and behavioral templates |
| **Agents** | Autonomous reasoning entities |
| **Tools** | Executable functions/capabilities |
| **Environments** | Task spaces and interaction contexts |
| **Memory** | Persistent state and learned representations |

Every resource has: unique ID, current version, interface signature, lifecycle state (created → validated → active → deprecated), and lineage back to its parent version.

### SEPL — Self-Evolution Protocol Layer

A closed-loop operator interface with three core operations:

1. **Proposal** — an agent suggests a modification to any RSPL-registered resource (refine a prompt, add a tool parameter, extend a memory schema)
2. **Assessment** — the proposal is evaluated against performance metrics and safety constraints
3. **Commitment** — validated changes are formally adopted as the new baseline; the previous version remains retrievable

## Auditable Lineage and Rollback

Every modification is timestamped, attributed to a proposing agent, and linked to the rationale that motivated it. This gives the protocol two safety properties that most self-improving systems lack:

- **Semantic provenance**: not just *what* changed but *why* — the decision rationale travels with the diff
- **Rollback**: if a change is later shown to degrade behavior (regression gating at the protocol level), the system reverts to any prior version of the affected entity

This is conceptually similar to `git` applied to agent internals, but richer — the unit of change is a typed resource modification, not a text diff.

## Evaluation

Tested on long-horizon planning and tool-use benchmarks; GPQA, GAIA, LeetCode, and DeepSeekMath-style tasks are referenced. The paper reports "consistent improvements over strong baselines" but does not publish a single headline number across all benchmarks.

No code repository is linked in the paper as of submission.

## Key Distinctions vs. Other Sources

| Dimension | AGP | Most other sources |
|-----------|-----|-------------------|
| Primary contribution | A **protocol** for managing self-modification | A specific optimizer, loop, or training algorithm |
| Unit of change | Typed, versioned resource modification | Text diff, code patch, weight update, prompt rewrite |
| Safety mechanism | Lineage + rollback at every operation | Regression gating, Pareto frontiers, held-out evals |
| Scope | Cross-cutting — applies to prompts, tools, memory, agents, environments | Usually one of: prompts, harness code, weights, architecture |

Related in spirit to [[sources/autoagent-kevinrgu]] (which defines a `program.md` / `agent.py` abstraction boundary for self-editing harnesses) and to [[sources/coral]] (which uses git + Attempts lineage for multi-agent coordination). AGP generalizes both: lineage is not a convenience but a protocol primitive.

## Relation to the Self-Improvement Loop

AGP maps cleanly onto the measure → fail → propose → gate → repeat cycle:

- **Measure / Fail** — happens outside the protocol; evidence is attached to proposals
- **Propose** — the Proposal operator; any agent can propose a change to any registered resource
- **Gate** — the Assessment operator; regression + safety checks run before commitment
- **Repeat** — Commitment updates the baseline; prior versions remain rollback targets

The novelty is not the loop itself but making the loop *inspectable* and *reversible* at every step. See [[concepts/regression-gating]] for how this compares with threshold and Pareto gating in other systems.

## Open Questions

- Does AGP's strict lineage model scale to thousands of proposals per run, or does the audit store become a bottleneck?
- How is the Assessment operator implemented — one shared evaluator, per-entity custom evaluators, or LLM-as-judge?
- The paper hints at "evolution-safe update interfaces" but doesn't detail how interface-breaking changes (e.g., a tool signature change) are handled by downstream callers.
- Without a reference implementation, it's unclear whether AGP becomes a standard (like MCP) or remains an academic framework.

## Connections

- [[concepts/self-improvement-loop]] — AGP formalizes the proposal/gate/commit phases as protocol operators
- [[concepts/regression-gating]] — Assessment + rollback is a generalization of threshold/Pareto gating
- [[concepts/knowledge-accumulation]] — lineage store is a form of session-level knowledge accumulation
- [[sources/autoagent-kevinrgu]] — defines its own abstraction boundary between declarative directive and executable harness; AGP generalizes this
- [[sources/coral]] — uses git lineage for multi-agent coordination; AGP proposes lineage as a universal primitive

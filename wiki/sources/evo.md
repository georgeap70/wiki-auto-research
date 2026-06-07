---
title: Evo — Autoresearch Orchestrator with Tree Search and Parallel Subagents
type: source
tags: [harness-optimization, autoresearch, tree-search, parallel, population, gates, rlm, gepa, evo-hq, karpathy]
sources: [evo-hq]
last_updated: 2026-06-06
---

# Evo

**Repo**: [github.com/evo-hq/evo](https://github.com/evo-hq/evo)
**Author**: Alok Kumar Bishoyi (evo-hq)
**License**: Apache-2.0

## One-line summary

An autoresearch orchestrator that turns any codebase into a self-optimizing loop: a `discover` skill instruments the benchmark, then parallel subagents in isolated git worktrees hill-climb against it under tree-search frontier strategies, with `gate` checks pruning regressions and RLM-inspired cross-cutting scans synthesizing failure patterns between rounds.

## Context

The repo is positioned as a productized version of the autoresearch idea Karpathy demoed: *"Inspired by Karpathy's autoresearch — where an LLM runs training experiments autonomously to beat its own best score."* Where [autoresearch-vs-hpo](autoresearch-vs-hpo.md) framed the abstract argument that an agent over the full code space beats classical HPO, Evo is one of the first packaged orchestrators that operationalizes that loop for an arbitrary repo and ships a dashboard, multi-backend execution, and gates as first-class primitives.

It composes inspirations from three lines of work already covered in the wiki:

- **Karpathy's autoresearch** — the overall "agent runs experiments to beat its own score" loop
- **GEPA** ([optimize_anything](optimize-anything.md)) — credited as the inspiration for the `pareto_per_task` frontier strategy that "keep[s] specialists the aggregate hides"
- **RLM** — *"RLM-inspired scan subagents"* read trace batches between rounds to surface cross-cutting failure patterns; the README cites arXiv 2512.24601

## The two-step workflow

```
/evo:discover   # one-time: explores the repo, picks what to measure, instruments evaluation
/evo:optimize   # runs the tree-search loop
```

`discover` interactively asks *what to optimize, the benchmark command, the metric direction*; the question can be skipped by seeding the answer:

```
/evo:discover make the JSON parser at src/parser.py faster
```

When `discover` builds a benchmark from scratch, it **automatically attaches a held-out-slice score-floor gate** — overfitting protection that the user does not have to remember to wire up. This is the same anti-overfitting primitive used by [Meta-Harness](meta-harness.md), but baked into the orchestrator's bootstrap step.

## Loop architecture: tree search over greedy hill-climb

Evo's distinctive design choice — explicit in its own framing — is **tree search rather than greedy hill-climbing**. The orchestrator maintains a tree of committed experiments; after each round, a *frontier strategy* selects which committed branch to extend. Multiple directions can fork from any committed node.

This sits between the single-thread sequential loops ([auto-harness](auto-harness.md), [Meta-Harness](meta-harness.md), [AutoAgent (KevinRGU)](autoagent-kevinrgu.md)) and the flat parallel populations of [EvoForge](evoforge.md) and [Group-Evolving Agents](group-evolve.md): Evo's parallelism is tree-shaped, not population-shaped, so it preserves lineage and lets the operator pick exploration vs. exploitation per round via the frontier knob.

### Frontier strategies (configurable)

| Strategy | Behavior |
|----------|----------|
| `argmax` | extend the highest-scoring branch |
| `top_k` | round-robin among the K best |
| `epsilon_greedy` | best most of the time, random sometimes |
| `softmax` | sample weighted by score |
| `pareto_per_task` | "keep specialists the aggregate hides" (GEPA-inspired) |

The presence of `pareto_per_task` is the load-bearing connection to [optimize_anything](optimize-anything.md): a scalar aggregate hides candidates that beat the rest of the field on specific tasks; Pareto-per-task selection retains them as branch parents.

### Parallel subagents

Within a round, subagents run in parallel. Each:
- Lives in *its own isolated workspace* (git worktree by default)
- *Picks up shared state* — failure traces, annotations, discarded hypotheses
- *Forms a hypothesis, edits, runs the benchmark*

Shared state is read at startup and written to during the round; this is a closer analog to [CORAL](coral.md)'s parallel-agent shared memory than to EvoForge's post-round synthesis. Discarded-hypothesis storage is unusual — most systems in the wiki keep only successful branches; Evo treats negative results as first-class signal, similar to [SkillOpt](skillopt.md)'s rejected-edit buffer (but at the granularity of *experimental hypotheses*, not text edits).

### Cross-cutting scans (RLM-inspired)

Between rounds:

> *"RLM-inspired scan subagents read trace batches in parallel and surface compound failure patterns: gate-failure intersections, shared root causes across traces. Findings land in shared state, which the next round's subagents read at startup."*

This is feedback compression in the [HALO](halo.md) sense: many traces → a smaller diagnostic signal. The differences from HALO are (a) Evo runs the scans *between rounds of its own loop*, not on production OpenTelemetry traffic, and (b) the scans look explicitly for *gate-failure intersections* — which gates fail together, which root causes recur across branches. This pushes the system away from per-experiment diagnosis toward systemic diagnosis.

## Gates as first-class primitives

> *"evo introduces gates: pass/fail checks that run on every experiment. An experiment that fails a gate is discarded even if its score beats the current best. Gates inherit down the experiment tree: a gate registered at the root runs on every descendant. Narrower gates can be attached to specific branches."*

Two properties worth flagging:

1. **Inheritance down the tree** — registering a gate at the root applies to every descendant. Branches can add narrower gates that apply only locally. This is the clearest mapping of regression-suite semantics onto a tree of experiments; the inheritance rule is the analog of [Autogenesis](autogenesis.md)'s versioned-resource lineage applied to safety constraints rather than to artifacts.
2. **Score-beating does not override a gate failure** — gates dominate the scalar reward. This is a stronger commitment than the [auto-harness](auto-harness.md) 80% threshold (which is a soft majority rule); Evo treats any gate as a hard veto.

Gates can be a test suite, an invariant script, or a score floor on a held-out slice of the benchmark. The held-out-slice gate is *auto-attached* during `discover`, which means even a naive user gets generalization protection by default.

## Execution backends

Evo decouples loop logic from where experiments execute:

- `worktree` (local, default)
- `pool` (local workspace reuse)
- `ssh` (custom SSH host)
- `modal` (Modal serverless)
- `e2b` (E2B cloud sandboxes)
- `daytona` (Daytona cloud workspaces)
- `aws` (AWS EC2 sandboxes)
- `azure` (Azure VMs)

This is one of the few systems in the wiki to package multi-backend execution as part of the harness rather than leaving it as integration work. Practical consequence: the same loop runs locally during development and on cloud sandboxes for an overnight production run, without rewriting the orchestrator.

## Dashboard

A web dashboard visualizes the experiment tree, configures the frontier strategy (per its "Frontier tab"), and surfaces gate failures. The dashboard makes the loop *inspectable* — closer in spirit to the auditable-lineage stance of [Autogenesis](autogenesis.md) than to the black-box overnight runs of early [auto-harness](auto-harness.md).

## Human-in-the-loop

Configurable via plain-language directives — the README documents the ability to *"pause after rounds or restrict to single experiments"* — otherwise the loop runs unattended. This is a softer HITL surface than [AutoAgent (HKUDS)](autoagent-hkuds.md)'s conversational driving but more present than EvoForge's batch runs.

## What is being optimized

Arbitrary code metrics discovered through repository analysis. The README's seed example optimizes the *speed of a JSON parser*; in general the system can optimize anything `discover` can identify as a benchmark — performance, resource efficiency, accuracy on a held-out task. Like [optimize_anything](optimize-anything.md) and [autoresearch-vs-hpo](autoresearch-vs-hpo.md), the scope is *"any code that maximizes a grader."*

## Feedback signal

Layered:

- **Per-experiment**: benchmark score (directional), gate pass/fail
- **Cross-experiment**: gate-failure intersections, shared root causes across traces (from cross-cutting scans)
- **Shared state**: failure traces, annotations, discarded hypotheses — accessible to every subagent at startup

The cross-cutting-scan layer is the distinguishing feature: most systems in the wiki feed back per-experiment traces (Meta-Harness, RLM-GEPA) or per-deployment trace batches (HALO). Evo runs scan subagents *as part of every round*, treating cross-trace pattern surfacing as a normal step in the loop rather than a one-shot analyzer.

## Loop architecture (per the [loop taxonomy](../concepts/self-improvement-loop.md))

A new variant worth naming: **tree-structured parallel hill-climb with shared-state cross-cutting scans**. Closer to [CORAL](coral.md) in shared-state mechanics, closer to [EvoForge](evoforge.md) and [Group-Evolving Agents](group-evolve.md) in parallel-population mechanics, but distinguished by (a) the tree-shaped exploration via configurable frontier strategies and (b) the explicit role of cross-cutting scans as a between-round phase.

## Connections

- [autoresearch-vs-hpo](autoresearch-vs-hpo.md) — Evo is one of the first packaged operationalizations of the autoresearch loop discussed there; explicitly credits Karpathy's autoresearch
- [optimize_anything](optimize-anything.md) — the `pareto_per_task` frontier strategy is GEPA-inspired; same Pareto-over-task-axes intuition transplanted into a frontier-selection role
- [HALO](halo.md) — cross-cutting scans are conceptually the same as HALO's specialized trace-analysis RLM (compress many traces into a diagnostic report), repurposed from production-traffic loops to between-rounds-of-tree-search
- [Meta-Harness](meta-harness.md) — both rely on held-out evaluation as the anti-overfitting gate; Evo automates the wiring during `discover`
- [auto-harness](auto-harness.md) — same broad "agent edits its harness against a benchmark" shape; Evo replaces the single-thread overnight loop with tree-shaped parallel subagents
- [EvoForge](evoforge.md) — both run parallel agent populations on harness optimization; EvoForge keeps a flat population, Evo keeps a tree with lineage; both share state across the population, though Evo also stores discarded hypotheses
- [Group-Evolving Agents](group-evolve.md) — both share experience across a parallel population; GEA uses a performance-novelty selection criterion, Evo uses configurable frontier strategies (one of which is `pareto_per_task`)
- [CORAL](coral.md) — shared-state mechanics most closely parallel CORAL's Attempts/Notes layer; Evo adds the discarded-hypothesis bucket
- [Autogenesis](autogenesis.md) — gate inheritance down the tree is the safety analog of versioned-resource lineage; both make the *audit trail* a structural property
- [SkillOpt](skillopt.md) — discarded-hypothesis storage parallels SkillOpt's rejected-edit negative buffer, at the granularity of experimental directions rather than text edits
- [RLM-GEPA](rlm-gepa.md) — both descend from the GEPA + RLM combination; RLM-GEPA optimizes skill instructions on a fixed RLM runtime, Evo uses RLM-inspired subagents as a trace-compression step inside a broader orchestrator
- [harness-optimization](../concepts/harness-optimization.md) — Evo is harness optimization with tree-structured parallelism and inheritable gates
- [evolutionary-optimization](../concepts/evolutionary-optimization.md) — tree search is the structured-branching cousin of flat population evolution; the frontier strategy is the selection operator
- [regression-gating](../concepts/regression-gating.md) — gate inheritance down the tree is a new gating primitive; auto-attached held-out-slice score floor on `discover` is a new bootstrap default
- [feedback-signals](../concepts/feedback-signals.md) — cross-cutting scans are between-round feedback compression analogous to HALO's RLM

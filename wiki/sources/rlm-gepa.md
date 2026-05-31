---
title: RLM-GEPA — Optimizing Recursive LM Skill Artifacts with GEPA
type: source
tags: [harness-optimization, gepa, rlm, recursive-lm, skill-instructions, dspy, agentspec, trampoline-ai]
sources: [rlm_gepa]
last_updated: 2026-05-30
---

# RLM-GEPA

**Repo**: [github.com/Trampoline-AI/predict-rlm](https://github.com/Trampoline-AI/predict-rlm) (subdirectory `src/rlm_gepa`)
**Author**: Trampoline AI (Gab Lespérance et al.); built on MIT CSAIL Recursive Language Models research (Alex L. Zhang, Tim Kraska, Omar Khattab)
**License**: MIT
**Install**: `pip install "predict-rlm[gepa]"`

## One-line summary

A production-grade optimization framework that uses [GEPA](optimize-anything.md) to evolve the **text components of a Recursive Language Model** (typically skill instructions) from scored execution traces — with an explicit `AgentSpec` declaring what behavioral changes are in-scope, so the optimizer doesn't have to infer project context.

## Context: what's an RLM?

predict-rlm (the parent project) is a "self-harnessed LM runtime that allows the LM to call its sub-lm with DSPy signatures." The root model never touches a long, messy conversational context — it interacts with the world through a **programmatic REPL**, issuing structured sub-calls via DSPy signatures. The runtime executes in a WASM sandbox with async tool support.

The motivation, in the repo's own words: *"a single line can represent 1M sub-calls — in direct contrast to agents like Claude Code that must mechanically emit each sub-agent call one at a time."* By staying inside a programmatic interface, the root LM is kept *"well within its comfortable operating range,"* avoiding the long-context degradation that plagues sequential agent harnesses.

RLM-GEPA is the subproject that **optimizes the text artifacts of an RLM** — primarily *skill instructions*, the prose-level descriptions of behavior that get injected into sub-calls.

## The two-loop architecture

```
Executor loop                                Proposer loop
─────────────                                ─────────────
1. Take current candidate (text components)  1. Read scored RunTrace objects
2. Run RLM on training examples              2. Analyze failure descriptions
3. Collect RunTrace + score + feedback       3. Propose surgical edits to text
4. Hand traces to proposer                   4. Hand new candidate to executor
```

The clean separation matches the **specialist optimizer/target** pattern seen in [HALO](halo.md), [AutoReason](autoreason.md), and [SkillOpt](skillopt.md): the agent producing proposals and the agent running them are distinct components.

## What's novel

### AgentSpec — declared optimization context

The `AgentSpec` is a structured description of *what kinds of behavioral improvements matter* for this project. It captures information GEPA cannot infer from traces alone:

- **Use cases** defining the transfer boundary (where the artifact must generalize)
- **Runtime affordances and grounding facts** (what tools and data exist)
- **Scoring methodology and feedback structure** (what the metric actually measures)
- **Counterfactual axes for generalization** (what variation the artifact must handle)

This is the framework's answer to a chronic problem in [optimize_anything](optimize-anything.md)-style text optimization: the optimizer has to guess at the surrounding product context, and without it, it overfits to incidental trace details. AgentSpec makes that context an explicit, typed input — closer in spirit to [Autogenesis](autogenesis.md)'s typed-resource protocol than to free-form prompt evolution.

### RLM as the source of truth

Projects subclass `RLMGepaProject` and derive their `agent_spec` via `agent_spec_from_rlm(...)`. The DSPy signatures and tools defined on the RLM are the canonical structure; GEPA only edits the *prose components* layered on top. This avoids the failure mode where the optimizer rewrites tool definitions or signatures into something that no longer compiles.

### Evidence-bounded feedback contract

The documentation states explicitly: *"optimization quality is bounded by the evidence your metric returns."* Effective feedback should *name specific failures* — missing findings, unsupported claims, extraction errors, wrong cells/clauses — rather than *prescribe text rewrites*. This is the exact same principle as [optimize_anything](optimize-anything.md)'s **Actionable Side Information (ASI)** and the [feedback-signals](../concepts/feedback-signals.md) thesis: describe the failure mode, let the proposer propose the fix.

### Surgical edits to skill instructions

Rather than full-prompt rewrites, the proposer issues targeted edits to "skill instructions" — the same unit treated as the "trainable state" in [SkillOpt](skillopt.md). Both projects converge on the idea that a structured prose artifact is the natural granularity for text-level optimization, and that *bounded* edits beat free-form regeneration. RLM-GEPA frames the constraint less explicitly than SkillOpt's "textual learning rate," but the intent is the same: don't overwrite working rules to fix a marginal failure.

## CLI surface

The framework exposes the typical optimization workflow as commands:

- `eval` — score the validation set with current text components
- `optimize` — run a GEPA search with budget controls (`--max-metric-calls` etc.)
- `stats` / `plots` — analyze the optimization artifacts produced by a run

## Recommended usage pattern

The README recommends driving the framework via a coding agent skill. The example invocation is itself instructive about the intended ergonomics:

> *"/rlm interview me to design a PredictRLM that extracts renewal terms... Then build the RLM, evals, and RLM-GEPA optimization wiring."*

The coding agent fills in the boilerplate: `seed_candidate()`, `load_trainset()`, `load_valset()`, `evaluate_example()`. The human contribution collapses to: define what the RLM should do, define what counts as success, and write the failure-description-style metric.

## Results

The X/Twitter announcement is paywalled (HTTP 402), so I haven't extracted concrete numbers. The repo positions this as production-grade infrastructure rather than a benchmark-driven research artifact — comparable to how [HALO](halo.md) is positioned as a deployable methodology rather than a paper with a results table.

## The RLM-in-HALO collision

There's a name-collision worth noting. [HALO](halo.md) uses the term "RLM" for a Recursive Language Model that has been specifically tuned for trace analysis — it sits between raw OpenTelemetry traces and the harness-editor. predict-rlm uses "RLM" in the original MIT CSAIL sense: a *general runtime* in which any LM can call sub-LMs via DSPy signatures.

These are not different things — they're the same underlying primitive used at different scales. HALO is "an RLM specialized as a trace compressor;" predict-rlm is "the framework for building any RLM, plus a way to optimize one." Both descend from the Zhang/Kraska/Khattab line of work. The wiki should treat "RLM" as a substrate, not as a specific system.

## What is being optimized

The text components of an RLM artifact — typically skill instructions, occasionally other prose-level behavioral fragments. The DSPy signatures, tools, and runtime structure are *fixed* during optimization; only the prose is mutated.

## Feedback signal

Rich and structured:

- **Score**: normalized 0.0–1.0, combining answer correctness with supporting-evidence quality
- **Feedback**: concrete failure descriptions (missing findings, unsupported claims, extraction errors) — explicitly **not** prescriptive rewrites
- **RunTrace**: full execution trace from the RLM runtime, available to the proposer as diagnostic context

This places RLM-GEPA at the rich-feedback end of the spectrum mapped in [feedback-signals](../concepts/feedback-signals.md), alongside [Meta-Harness](meta-harness.md), [HALO](halo.md), and [optimize_anything](optimize-anything.md).

## Loop architecture (per the [loop taxonomy](../concepts/self-improvement-loop.md))

Closest to the **specialist optimizer/target separation** family — executor and proposer are distinct, with the RLM playing the role of the (fixed) target.

## Connections

- [optimize_anything](optimize-anything.md) — RLM-GEPA *is* GEPA applied to RLM text artifacts. The Pareto-mutation-selection primitive is identical; AgentSpec is the new layer
- [SkillOpt](skillopt.md) — both treat skill instructions as the unit of optimization; both use surgical/bounded edits; both report substantial transfer across deployment contexts. SkillOpt's "textual learning rate" and RLM-GEPA's "surgical edits" are the same intuition framed differently
- [HALO](halo.md) — both rely on a specialized component sitting between raw execution data and the editor; both adopt the MIT-CSAIL RLM primitive (HALO trains one for trace analysis, predict-rlm provides the runtime)
- [Meta-Harness](meta-harness.md) — both use full execution traces as the feedback channel; Meta-Harness edits harness code, RLM-GEPA edits skill prose layered on top of a structured runtime
- [Autogenesis](autogenesis.md) — AgentSpec is a typed, declared description of what's in-scope for modification, akin to Autogenesis's typed-resource protocol but applied to optimization context rather than to the resources themselves
- [auto-harness](auto-harness.md) / [HonedHaiku](honedhaiku.md) — share the same outer pattern (production-like rollouts → diagnostic feedback → text edits) at different scales of artifact
- [harness-optimization](../concepts/harness-optimization.md) — RLM-GEPA sits at the **structured skill-instruction sub-layer** of harness optimization (a layer up from raw system prompts; a layer below full-harness code editing)
- [feedback-signals](../concepts/feedback-signals.md) — the evidence-bounded feedback principle (*"name failures, don't prescribe rewrites"*) is a clean restatement of the ASI thesis from optimize_anything
- [knowledge-accumulation](../concepts/knowledge-accumulation.md) — skill instructions are the persistent, accumulated knowledge artifact; transfer across use cases is the stated design goal

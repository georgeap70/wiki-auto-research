---
title: "SKILL.state: Scalable Long-Horizon Agent Skills"
type: source
tags: [runtime-architecture, context-engineering, execution-state, long-horizon, token-efficiency, context-poisoning, state-tracking, no-weight-updates]
sources: [skill.state]
url: https://arxiv.org/abs/2608.26263
code: none
authors: Sanket Badhe, Priyanka Tiwari, Jonghyun Chung
affiliations: Google LLC, Purdue University
arxiv: 2608.26263
last_updated: 2026-09-08
---

# SKILL.state (arXiv 2608.26263)

## Summary

SKILL.state is a **runtime architecture**, not an optimization loop — the odd one out among this wiki's harness sources, and a useful one. It makes a single structural change: replace the append-only conversational history that every mainstream agent runtime maintains with an **explicit, mutable execution state**, and **discard the intermediate reasoning trace immediately** after it has produced a validated state update.

At each step the model receives exactly three things:

```
A_t = (P, Σ_t, O_t)
```

- `P` — the **immutable** procedural specification (the skill)
- `Σ_t` — the current **structured execution state**
- `O_t` — the **latest** environment observation

The model never sees previous observations, previous actions, or previous reasoning. The consequence is a **strictly bounded O(1) prompt footprint** and **O(T) cumulative token cost**, against O(t) and O(T²) for conversational runtimes.

The results say this is not merely a cost optimization: bounded, structured state **also improves accuracy**, and by a widening margin as horizons grow.

## The Mechanism

The model generates a triple `(R_t, ΔΣ_t, a_t)`: a chain-of-thought reasoning trace, a structured state update (a JSON dictionary of key mutations and deletions), and the action to execute. State advances by

```
Σ_{t+1} = Σ_t ⊕ ΔΣ_t
```

where `⊕` is the runtime's dictionary-merge operator with **null-deletion semantics**. Critically, **within-step multi-step reasoning is fully intact during generation** — the architecture does not sacrifice deductive planning. It only refuses to *carry the reasoning forward*: once the transition is validated and applied, `R_t` is discarded permanently.

**Schema ownership sits in the deterministic runtime, not the model.** Malformed model output therefore cannot corrupt persistent state — an invalid patch triggers a rollback-retry cycle. Schemas are authored **once per domain**, not per task: across all 100 InterCode CTF instances the agent reuses one static 5-field schema (`discovered_flags`, `tested_hypotheses`, `active_files`, `working_dir`, `cmd_summary`).

The paper distinguishes itself carefully from three neighbors. **Memory systems** (summarization, episodic retrieval, persistent stores) alleviate context growth but preserve conversational semantics — future decisions still condition on textual reconstructions of the past. **LangGraph-style frameworks** inject auxiliary structured state but keep the transcript as the primary reasoning substrate. **Dialogue State Tracking** maintains structured slots *alongside* full transcripts in quasi-static dialogue; SKILL.state treats the structured state as a **sufficient statistic** and throws the transcript away.

## Evaluation

**SkillExecBench** (introduced here) — two controlled environments with deterministic ground-truth transitions:
- *Warehouse Management*: 500 independent shelves; Store/Ship/Move/Wait. Tests maintaining many independent, non-overlapping variables past the point where early observations leave the context window.
- *Software Repository*: a nested relational graph of branches, commits, PRs, CI statuses; CherryPick/Merge/RunTests/CreateRelease/Rollback. Dense dependencies — merging a PR alters the target branch and dependent PRs — testing structural reasoning over an entangled graph.

**Public benchmarks** — InterCode CTF (100 Linux bash CTF challenges) and Sierra τ-Bench (Retail, Airline).

**Baselines** — three runtime paradigms (Prompt/ReAct; Memory/summarization with a rolling 3-step window; Stateful/LangGraph-style) plus three budget-matched compression controls (sliding-window truncation, summary-capped, ReAct + LLMLingua).

Models: **Gemini-3-Flash, Gemma-4-31B-it, Qwen-3-8B-it**; temperature 0.0, top-p 1.0; 5 procedural generator seeds; differences at T ≥ 50 significant by paired t-test (p < 0.01).

## Results

### Long-horizon scaling (Warehouse, Gemini-3-Flash)

| T | Runtime | Score | Avg prompt (chars) | Total tokens |
|---|---------|-------|--------------------|--------------|
| 50 | ReAct | 0.88 | 11,931 | 171,658 |
| 50 | **SKILL.state** | **0.96** | **1,773** | **30,151** |
| 100 | Stateful (LangGraph) | 0.91 | 31,354 | 1,062,387 |
| 100 | **SKILL.state** | **0.94** | **1,905** | **65,408** (**16.2×** fewer) |
| 200 | ReAct | 0.74 | 48,007 | 2,608,755 |
| 200 | Memory | 0.84 | 84,364 | 6,175,509 |
| 200 | Stateful | 0.88 | 72,305 | 5,041,164 |
| 200 | **SKILL.state** | **0.94** | **1,811** | **122,384** |

The prompt size is flat (~1,800 chars) across a 20× horizon increase. Baseline accuracy **decays** with horizon (ReAct 0.90 → 0.74); SKILL.state holds at 0.94.

### Public interactive benchmarks (Gemini-3-Flash)

| Runtime | InterCode CTF pass@1 | τ-Bench Retail | τ-Bench Airline |
|---------|----------------------|----------------|-----------------|
| ReAct | 43.2% | 48.2% | 21.8% |
| Memory | 46.4% | 29.9% | 23.6% |
| Stateful | 41.8% | 51.7% | 28.1% |
| **SKILL.state** | **54.2%** | **58.3%** | **32.4%** |

Highest success on all three *and* lowest token cost. On CTF, keeping hypotheses and discovered flags in `Σ_t` prevents the model from repeating failed commands: **+7.8 points over the strongest baseline**, with tokens down 60.4% vs. ReAct and 65.9% vs. Stateful. On τ-Bench Airline, where complex database responses push baseline prompts above 11,000 tokens/step, SKILL.state holds ~2,800 tokens/step.

### Noise robustness (T = 50)

Distractor events (system telemetry, irrelevant git activity, rule overrides) injected at 5 / 20 / 50 per turn:

| Runtime | 5 events | 20 events | 50 events |
|---------|----------|-----------|-----------|
| Prompt (ReAct) | 0.68 | 0.61 | 0.53 |
| **SKILL.state** | **1.00** | **0.97** | **0.98** |

Distractors are filtered out during state-patch generation and **never enter subsequent prompts** — the mechanism is structural, not a matter of the model resisting distraction better.

### State recovery — the cleanest result

When the true world state is modified *outside* the agent's action loop (an external actor moves an inventory item), history-based baselines **hallucinate for 5–8 consecutive turns**, because obsolete facts in the prompt history overpower contradictory new observations. SKILL.state requires **zero recovery steps**: decisions depend on current structured state, so a corrective alert lands immediately.

This is [context poisoning](../concepts/context-engineering.md) demonstrated as a measurable, quantified failure mode, and it applies to any long-running agent whose environment can change beneath it. (One scenario, a canceled order, defeats every runtime including SKILL.state.)

### Budget-matched controls — structure, not brevity

Pinning every baseline to SKILL.state's ~1,800-token budget (Warehouse, T = 100):

| Configuration | Score |
|---------------|-------|
| Full ReAct (unbounded, 1.25M tokens) | 0.84 |
| Truncated (sliding window) | **0.18** |
| Summary-capped | 0.52 |
| ReAct + LLMLingua | **0.22** |
| **SKILL.state** | **0.94** |

This is the ablation that makes the paper's case. **The gain is not from shorter prompts.** Sliding-window truncation collapses because critical early inventory allocations are evicted; LLMLingua collapses because statistical entropy filtering removes seemingly-redundant slot identifiers that are semantically vital. Structured state preserves exact relational dependencies that statistical compressors destroy — a strong statement that *what* you keep matters more than *how much*.

### Where it breaks: open-weight models

On Gemma-4-31B at T = 100 the score is 0.42 (still equal to Stateful and 2× ReAct's 0.21, at 1/14 the tokens). The failure taxonomy is the useful part:

| Error mode | Share |
|------------|-------|
| Premature state overwrite / deletion (omitting existing keys instead of merging in place) | **68%** |
| Schema comprehension / type coercion | 20% |
| JSON syntax / formatting slips | 12% |

**Small-model degradation stems from structured-output adherence, not reasoning capacity** — which points at grammar-constrained decoding as the fix rather than a better model.

## What Is Optimized / Feedback / Gating

| Axis | SKILL.state |
|------|-------------|
| Optimization target | **Nothing is searched** — this is a fixed runtime architecture, hand-designed once per domain (the schema) |
| Feedback signal | None in a learning sense; the runtime *validates* each state patch deterministically |
| Loop structure | Not a self-improvement loop — an inner-loop execution substrate that other loops run on top of |
| Gating | Deterministic schema validation per step, with rollback-retry on invalid patches |
| Human involvement | Author the domain schema once (5 fields sufficed for 100 CTF tasks) |

## Why It Matters (and why it's in this wiki)

SKILL.state doesn't self-improve, so it belongs here for three reasons:

1. **It is the substrate the other loops optimize.** Every harness in the wiki contains a context-management strategy. SKILL.state argues that the standard choice — append everything — is quadratically wasteful *and* actively harmful at long horizons, and that fixing it structurally beats optimizing around it. It is directly relevant to the [`θ_middleware` block](autosaddler.md) that [AutoSaddler](autosaddler.md) patches and the **C**ontext and **S**tate modules that [HarnessDev](harnessdev.md) requires creators to implement.

2. **It inverts the wiki's dominant assumption about accumulated context.** [Knowledge accumulation](../concepts/knowledge-accumulation.md) is this wiki's central compounding mechanism — *keep more, structured better*. [ACE](ace.md) is explicitly designed against erosion of accumulated detail; [WikiSkill](wikiskill.md) never resets its wiki. SKILL.state says: within a single execution, **aggressive forgetting is correct**, provided the state is a sufficient statistic. These are not in conflict once separated by time horizon — accumulate *across* runs, discard *within* one — and stating that boundary is the source's main conceptual contribution. It also sharpens ACE's context-collapse concern: collapse is bad when a *rewrite* silently erases detail; deliberate projection into a validated schema is not the same operation.

3. **Its limitations name the conditions for the whole approach.** The state must be a sufficient statistic for future execution, which fails in three settings the paper is candid about:
   - **No fixed schema known in advance** — the relevant state structure must be discovered during execution. (This is exactly the gap a [skill-evolution loop](wikiskill.md) could fill: evolve the schema.)
   - **Delayed relevance** — a correct update depends on an earlier observation whose relevance wasn't recognized when first seen, so it was never committed to state. Unlike a transcript, discarded context cannot be revisited.
   - **History *is* the objective** — auditing, debugging provenance, explaining past actions. Here the trajectory is the target output, not overhead. Note this directly conflicts with the [lineage-and-rollback safety substrate](autogenesis.md) that [Autogenesis](autogenesis.md) argues for and that let [DGM](dgm.md) detect its own reward hacking.

   Also single-agent only: a shared execution state is a natural multi-agent coordination substrate (cheaper than exchanging quadratic transcripts, à la [CORAL](coral.md)'s shared stores), but concurrent writes would require deterministic conflict-resolution semantics in `⊕` that the single-agent setting never exercises.

An observation worth flagging: [HarnessDev](harnessdev.md) found that **no checkpoint event appeared in 26,679 trajectories** of agent-built harnesses, and that every never-triggered component it audited concerned state and memory. SKILL.state argues explicit execution state is the highest-leverage runtime abstraction available; HarnessDev shows frontier models will not build it unless told to. Together they identify a concrete, high-value gap that current harness-optimization loops are not filling.

## Connections

- [concepts/context-engineering](../concepts/context-engineering.md) — a third position beyond ACE's content and MCE's mechanism: *replace* the accumulating substrate with validated state
- [concepts/knowledge-accumulation](../concepts/knowledge-accumulation.md) — the counter-case; separates within-run discarding from across-run accumulation
- [concepts/harness-optimization](../concepts/harness-optimization.md) — the context/state layer of the harness, hand-designed rather than searched
- [sources/ace](ace.md) — context collapse and the anti-erosion discipline, in productive tension with deliberate discarding
- [sources/wikiskill](wikiskill.md) — the complement: SKILL.state fixes the runtime, WikiSkill evolves the specification `P` that runs on it
- [sources/harnessdev](harnessdev.md) — evidence that agent-built harnesses systematically omit exactly this mechanism
- [sources/autosaddler](autosaddler.md) — middleware/agent-loop patches are where a loop would discover this kind of change
- [sources/autogenesis](autogenesis.md) — the auditability tension: discarded reasoning is unauditable reasoning

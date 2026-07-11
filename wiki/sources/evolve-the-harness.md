---
title: "Don't Train the Model, Evolve the Harness (Harvey's LAB)"
type: source
tags: [harness-optimization, meta-harness, legal-agents, deterministic-code, transfer, regression-gating, mismanaged-geniuses]
sources: [hf-harness]
url: https://huggingface.co/spaces/joelniklaus/harness-optimization
code: https://huggingface.co/spaces/joelniklaus/harness-optimization
authors: Joel Niklaus
last_updated: 2026-07-10
---

# Don't Train the Model, Evolve the Harness (Joel Niklaus)

## Summary

A worked application of the [Meta-Harness](meta-harness.md) loop to a hard, realistic domain: **legal agents on Harvey's Legal Agent Benchmark (LAB)**. The weights are frozen; an automated loop rewrites only the runtime scaffold around the model and lifts it **+20 points** on LAB's pooled criterion metric. The framing is the *"mismanaged-geniuses hypothesis"*: capable models underperform because of brittle surrounding infrastructure, not fundamental reasoning limits.

Its most cited-worthy contribution is empirical and blunt: **deterministic code mechanisms, not prompt edits, drive the gains — and code transfers across model families while prompt playbooks do not.**

## Setup

- **Fixed agent model:** DeepSeek-V4-Pro (served via Hugging Face Inference Providers, OpenAI-compatible router). No fine-tuning; only the harness changes.
- **Proposer (optimizer):** Claude Opus 4.8, run via a Claude CLI subscription for long, unmetered proposer sessions with auto-resume.
- **Judge:** Claude Sonnet 4.6 scoring outputs against detailed LAB rubrics.
- **Benchmark — LAB:** agents get a closed workspace of documents (contracts, emails, spreadsheets, decks) and must produce deliverables (memos, issue lists, redlines) saved to an *exact* filename/directory or the task scores zero. Two metrics: **pooled criterion pass rate** (dense, the optimization target) and **all-pass rate** (whole-task success, the strict headline metric).
- **Splits** (from 1,251 public LAB tasks): **dev = 24 tasks** (one per practice area, drives the loop); **test = 100 tasks** held out (hardest cases — up to 1.5M-token context, 11 deliverables, 194 criteria, 55 documents). Zero overlap.
- **Baselines:** vanilla LAB harness 63.1% pooled / 0% all-pass on dev. Open-source agent harnesses trail badly: Pi 45.4%, Goose 23.2%, mini-swe-agent 3.5% pooled.
- **Cost:** a full 100-task test run ≈ $120–160.

## Method

A single-frontier ("compounding-line", not population) [Meta-Harness](meta-harness.md) loop:

1. **Proposer** reads run history and frontier lineage, forms a hypothesis, and proposes **exactly one new mechanism**, writing `pending_eval.json`.
2. **Evaluator** (`meta_harness.py`) runs the candidate over 24 dev tasks × 3 trials with a deterministic scaffold; the judge scores it without touching the held-out split (`_touched_test()` guard).
3. **Promotion**: candidate becomes the new best iff its blended score beats the incumbent by ≥1 point (just above 3-trial noise).

**Blended score:** `pooled_criterion_rate + 0.5 * all_pass_rate − 0.005 * tokens_per_million` — a dense primary target, an all-pass bonus to prevent luck-based wins, and a token-efficiency penalty.

**Guardrails** (see [regression gating](../concepts/regression-gating.md)):
- **Copy-and-adapt**: each candidate begins as an exact copy of the best harness, so accepted mechanisms are inherited and never silently lost — wins compound.
- **Causal-replay / pooled validation**: a mechanism must prove itself either by re-scoring deterministic fixes on old transcripts, or by pooled comparison across ≥5 fix tasks and ≥5 regression tasks. Single-trial swings are rejected as noise.
- `loop_state.json` maintains continuity across restarts.

## Results

| Metric (dev) | Vanilla | Final frontier | Gain |
|--------------|---------|----------------|------|
| Pooled criterion | 63.1% | 83.3% | **+20.2** |
| All-pass (whole-task) | 0% | 4.2% | +4.2 |

Held-out **test**: pooled 63.4% → 80.1%; all-pass 0% → 5.0% — the gains generalize off the dev set. Final positioning sits between Claude Sonnet 4.6 and Opus 4.6 on LAB's headline metric — a frozen open model scaffolded up into frontier-proprietary territory. ~19 candidates were evaluated.

### The key findings

- **Code beats prompts.** *Five of the top six harnesses are deterministic code, not prompt edits.* The single largest gain (`deliverable_landing_gate`, dev pooled 72.8% → 79.2%) simply guarantees files land in the correct location with exact names. Four mechanism families emerged: deliverable landing/delivery (code, zero extra model tokens), matter fidelity (`matter_audit_allwork` + small best-of-N vote), loop robustness (`toolcall_json_repair`, repetition-loop breaks), and prompt playbooks (early wins, then stalled).
- **The loop fixes operational failures, not reasoning.** Gains came from deliverables in the wrong place, provider-corrupted tool calls, repetition loops, and drafts about the wrong matter — not from making the model reason better. Substantive-quality gains plateaued.
- **Transfer is asymmetric.** Applying the DeepSeek-V4-Pro-optimized harness to other models: V4 Flash (same family) gained **+14.4**, but Nemotron-3 Ultra (different family) gained only **+0.4**. Code mechanisms transferred; prompt playbooks were model-specific and sometimes *degraded* other models. This is the transfer-side complement to [Self-Harness](self-harness.md)'s "harness edits are model-specific" claim — with the refinement that *deterministic-code* edits transfer far better than *prose* edits.

## Limitations & future work

Only 24 dev tasks (may miss failure modes); prompting alone was weak; the rubric judge is imperfect; substantive-expertise tasks plateaued. Proposed next steps: cheaper judges (V4 Flash ≈ 21× cheaper input) to afford larger dev sets, more domains (physics, critical-thinking), population diversity instead of a single frontier, and specialized tools (web search, spreadsheet parsing).

## Connections

- [concepts/harness-optimization](../concepts/harness-optimization.md) — a domain application of Meta-Harness; strongest wiki evidence that *deterministic code* is the high-transfer harness component
- [sources/meta-harness](meta-harness.md) — the framework this instantiates (single compounding frontier variant)
- [sources/self-harness](self-harness.md) — converging (harness slack is real) and contrasting (code transfers, prompts don't) on model-specificity
- [concepts/regression-gating](../concepts/regression-gating.md) — copy-and-adapt inheritance + ≥1-point promotion + causal-replay validation
- [concepts/feedback-signals](../concepts/feedback-signals.md) — blended dense+bonus+cost score as the acceptance signal
- [sources/weng-harness-blog](weng-harness-blog.md) — Weng's survey frames exactly this "evolve the scaffold, freeze the model" thesis
- [sources/skillopt](skillopt.md), [sources/alphaevolve](alphaevolve.md) — related transfer / evolutionary-frontier framings referenced in the writeup

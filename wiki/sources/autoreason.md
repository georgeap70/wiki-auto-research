---
title: AutoReason — Tournament-Based Self-Refinement
type: source
tags: [self-refinement, tournament, borda-count, critique, judging, nous-research]
sources: [autoreason]
last_updated: 2026-04-25
---

# AutoReason

**Repo / Paper**: [github.com/NousResearch/autoreason](https://github.com/NousResearch/autoreason) (paper at `paper/autoreason.pdf`)
**Authors**: SHL0MS · HERMES AGENT (Nous Research)

## One-line summary

Replaces the naive *critique → revise* self-refinement loop with a per-iteration tournament between three candidates (incumbent, adversarial revision, synthesis) judged by fresh blind voters via Borda count — fixing three structural failures of single-agent critique loops.

## The problem with vanilla self-refinement

AutoReason identifies three pathologies in the standard "ask the model to critique its output, then revise it" pattern:

| Failure | What goes wrong |
|---------|----------------|
| **Prompt bias** | When asked to critique, models hallucinate flaws regardless of quality |
| **Scope creep** | Outputs grow unboundedly across iterations as each revision adds material |
| **Lack of restraint** | Models never decide "no change needed" — they always edit something |

These compound: each iteration injects fabricated criticism, which produces fabricated improvements, which expand without bound.

## The tournament solution

Each iteration produces three candidates from fresh, context-isolated agents:

| Candidate | How it's produced |
|-----------|-------------------|
| **A** — Incumbent | Unchanged from previous round |
| **B** — Adversarial revision | Critic writes critique → Author writes revision |
| **AB** — Synthesis | Synthesizer combines A and B |

A **judge panel** (fresh agents with no shared context) ranks the three blindly; **Borda count** aggregates the rankings. Winner becomes the new incumbent. **Convergence**: when the incumbent wins two rounds in a row.

This is a *gated* loop: change only happens when independent judges prefer the changed version, eliminating the "always revise" bias.

## What is being optimized

The output itself — essays, code, reasoning — not the model, prompt, or harness. AutoReason is an **inference-time self-improvement** loop, applied per query rather than across queries.

## Key results

| Setting | Result |
|---------|--------|
| Haiku 3.5 on 3 writing tasks | Perfect sweep (wins all comparisons) |
| Sonnet 4.6 on CodeContests (150 problems) | 77% vs 73% baseline |
| vs best-of-6 sampling, matched compute | 40% vs 31% — same budget, better outcome |
| Word-count-controlled comparison | Wins 21/28 — gains aren't from longer outputs |

### Diminishing returns
Haiku 4.5 transition point at 60% accuracy: above that, incremental gains shrink. The technique is most effective on models in the productive band (similar in spirit to the [[sources/honedhaiku|Goldilocks band]]).

## Ablations

- **7 judges converge 3× faster than 3 judges** — judge count materially affects convergence speed
- **Both B and AB are necessary**: removing the adversarial revision *or* the synthesis collapses performance. The synthesis acts as a hedge against bad revisions; the adversarial component is what introduces real change
- Tournament gating is what makes the loop terminate cleanly — without it, scope creep continues

## Connections

- [[concepts/self-improvement-loop]] — AutoReason is a per-query inference-time loop; complementary to per-deployment harness loops
- [[concepts/regression-gating]] — the Borda tournament *is* the gating mechanism; voters must prefer the change for it to land
- [[concepts/feedback-signals]] — uses peer judgment (multi-agent voting) as the signal rather than execution traces or scalar rewards
- [[sources/honedhaiku]] — both observe a productive band where optimization works; AutoReason's diminishing returns at 60%+ echoes HonedHaiku's 50–70% Goldilocks band
- [[sources/coral]] — both leverage multi-agent independence (fresh contexts, blind judging), though for different purposes (CORAL: parallel exploration; AutoReason: bias-free judgment)

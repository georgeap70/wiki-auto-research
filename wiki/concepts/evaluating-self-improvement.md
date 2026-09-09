---
title: Evaluating Self-Improvement
type: concept
tags: [evaluation, benchmark, held-out, generalization, overfitting, noise-floor, creator-executor-separation, efficiency, negative-result]
sources: [harnessdev, auto.saddler, wiki.skill, skill.state, interaction-trajectory-mining, weng-blog, hf-harness, self-harness, skillOpt, ophis, dgm, stop]
last_updated: 2026-09-08
---

# Evaluating Self-Improvement

Nearly every source in this wiki reports the same shape of result: *baseline X, after-optimization Y, gain Y−X*. This page is about the question that shape hides — **under what conditions is a reported gain real, durable, and attributable to the change that was made?**

It became a distinct subject in 2026, when a cluster of benchmarks arrived whose object of study is the improvement loop itself rather than any task. [HarnessDev](../sources/harnessdev.md) is the wiki's anchor source; [Weng's survey](../sources/weng-harness-blog.md) had already named "weak evaluators" as the field's first open challenge.

## Why a Gain Can Be Unreal

Five distinct failure modes, each documented by a source here:

### 1. The gain is inside the noise floor

The single most under-reported number in this literature is the variance of re-running the *same* artifact. [HarnessDev](../sources/harnessdev.md) measured it: the same frozen commit varies by about **±4.75 pair-score points**. Of its 64 official version switches, **27 reported gains that fall inside that band** and only **2 had clear positive evidence beyond it**. Small reported gains cannot be attributed to code changes from score alone.

Systems that take this seriously build noise-awareness into the accept rule rather than the write-up:

| System | Noise discipline |
|--------|------------------|
| [evolve-the-harness](../sources/evolve-the-harness.md) | **≥1-point promotion threshold**, set just above the noise floor of 3-trial averaging |
| [OPHIS](../sources/ophis.md) | **10 repeated evaluations**, must clear **≥3σ** over baseline; high-mean-but-unstable interventions rejected |
| [AutoSaddler](../sources/autosaddler.md) | Three repeated test runs, reported as mean ± SD |
| [WikiSkill](../sources/wikiskill.md) | Three independent full-evolution runs; paired bootstrap significance |
| [AFlow](../sources/aflow.md) | Each workflow executed 5× during evaluation |

### 2. The gain doesn't survive held-out tasks

The gap between a feedback-set gain and a held-out gain is the clearest single diagnostic of overfitting, and it is large. [HarnessDev](../sources/harnessdev.md):

| Lineage (self-runtime) | Feedback-set gain | Held-out gain |
|------------------------|-------------------|---------------|
| Qwen 3.7 Max | +13.9 | +1.43 |
| DeepSeek V4 Pro | +13.4 | +3.17 |
| Gemini 3.1 Pro | +8.8 | +2.70 |
| GPT-5.5 | +5.9 | +3.81 |
| Opus 4.8 | +3.0 | +4.44 |

Feedback-set gains shrink 2–4× — and *rank differently*. Worse, across 64 switches, **feedback and held-out scores moved in the same direction only 34 times (53.1%)**, and only **2 of 9** creator-declared final versions were the held-out-optimal version of their own lineage. The paper's conclusion is the crisp form of the lesson: *visible feedback is useful for local search but unreliable for final selection.*

[AutoSaddler](../sources/autosaddler.md) ablates the fix. Removing generalization-aware selection (reflection + dev-set evaluation) is its **largest single ablation drop, 62.0 → 50.6** — and the decomposition explains why: **fix rate is essentially unchanged** between the two settings, while the **regression rate diverges** (−0.24 pp/iter with the gate, +0.16 pp/iter without). Proposal quality and durability are different problems.

The structural answer is split design:

- **Disjoint task *groups*, not random splits.** [AutoSaddler](../sources/autosaddler.md) trains on qutebrowser, validates on Vuls + NodeBB, and tests on Ansible + Flipt + Element-web — so the metric measures cross-repository generalization, not memorization of a repo's quirks.
- **Held-out scores never shown to the loop.** [HarnessDev](../sources/harnessdev.md) evaluates every frozen version on its 630-task held-out split *after* all trajectories end.
- **Automatic held-out gates.** [Evo](../sources/evo.md) auto-attaches a held-out-slice score-floor gate during `discover`, so generalization protection is a default rather than something a user must remember.
- **Leak guards.** [evolve-the-harness](../sources/evolve-the-harness.md)'s `_touched_test()` and [AutoSaddler](../sources/autosaddler.md)'s exposure of only functional-logic source files (evaluation and benchmark-data code withheld from the patching agent).

### 3. The improvement is credited to the wrong component

If the same model both builds the harness and runs inside it, a reported gain conflates harness quality, executor capability, and the compatibility between them. [HarnessDev](../sources/harnessdev.md)'s central design move is to **separate the creator LLM from the executor LLM**:

```
(L_C, D) → H          # creator builds harness in dev environment
(H, L_E, x) → y → score   # H frozen; executor runs it
```

and to report both **Self-Eval** (`L_E = L_C`) and **Unified-Eval** (fixed executor). The spread is not marginal: Opus's SWE-Pro harness scores **69.3 under itself and 33.0 under Gemini**, while Qwen's harnesses *gain* +17.6 (BrowseComp) and +12.9 (MLE-bench) under Gemini — its own executor had been the bottleneck. One Opus harness hard-codes a **120-step limit** tuned to its original executor; its Search harness's duplicate-query rate rises **10.1% → 88.2%** when the executor changes.

[WikiSkill](../sources/wikiskill.md) reaches the same separation from the skill side and states it as a capability claim: **self-evolution conflates *discovering* useful procedural knowledge with *executing* it.** Its evidence is a case where a model authors knowledge more useful to another model than to itself — Qwen-3.5-4B's OfficeQA skills *hurt* Qwen-3.5-4B (30.2 → 28.5) but *help* Qwen-3.6-27B (42.1 → 52.9).

Implication for reading every other source here: a single-model self-improvement result (e.g. [Self-Harness](../sources/self-harness.md)) measures the creator–executor *system*, and cannot by construction separate the two.

### 4. The score was obtained through a route the metric didn't intend

This is [reward hacking](regression-gating.md), documented in [STOP](../sources/stop.md) (sandbox bypass; >1000% spurious "accuracy") and [DGM](../sources/dgm.md) (faked test logs; deleting the markers its own hallucination detector used).

[HarnessDev](../sources/harnessdev.md) contributes the constructive counterpart — an **audit protocol with a clean null result**. Two properties make its constraints checkable rather than advisory:

1. **The score path is isolated from the artifact.** A harness's self-reported status is never a scoring input; SWE-Pro credit comes only from the real repository diff, Terminal-Bench credit only from final environment state. *No harness can earn score by asserting success.*
2. **Every run retains trajectory, result, and metric artifacts alongside the frozen source**, supporting a post-hoc audit of both the delivered code and what that code actually executed.

Every run in the paper was audited; no harness obtained score through a prohibited route. [WikiSkill](../sources/wikiskill.md) applies the same instinct at a smaller scale: `skill-impact.md` is written **programmatically by the outer harness**, not by any LLM, so the accept/reject record the Proposer reads is ground truth rather than self-report.

The generalizable rule: **make the metric unassertable by the agent, and keep the artifact separate from the record of what it did.**

### 5. The change never actually ran

The most easily overlooked failure mode, and one only artifact inspection catches. [HarnessDev](../sources/harnessdev.md) audited generated harnesses rather than only scoring them:

- Of 108 Code component instances, **18 never trigger in any run** — all of them state/memory. **No checkpoint event appears in 26,679 recorded trajectories.**
- **124 of 587 Writing features are confirmed dead code**; 36 Data mechanisms sit on dead paths.
- Of 169 functions/classes added during Evolution, 113 are reachable, 31 only through dead code, and **25 have no caller at all**.
- One official version switch **contained no executable code change**.

A loop can therefore report a "gain" from a mechanism that has never executed. The corollary is that **edit volume is not evidence**: Gemini added the fewest lines (1,006) yet scored best on Terminal-Bench. Self-test *count* correlates with downstream score at only ρ = 0.13–0.26 (not significant), while **revision calls reach ρ = 0.57 (p ≤ .0005)** — testing pays only when the creator reads the failure, changes something specific, and re-verifies.

## What Should Be Reported

Synthesizing across sources, an adequate self-improvement result carries:

| Quantity | Why | Best current practice |
|----------|-----|----------------------|
| Re-run variance of the fixed artifact | Otherwise gains are unreadable | [HarnessDev](../sources/harnessdev.md) (±4.75), [OPHIS](../sources/ophis.md) (10 evals, 3σ) |
| Held-out score, split by task *group* | Separates search from generalization | [AutoSaddler](../sources/autosaddler.md), [HarnessDev](../sources/harnessdev.md) |
| Score under a *different* executor model | Separates artifact quality from co-adaptation | [HarnessDev](../sources/harnessdev.md) Unified-Eval; [AutoSaddler](../sources/autosaddler.md) Opus→Haiku (+5.6pp) |
| Rollout / trace budget consumed | Gains are meaningless without cost | [AutoSaddler](../sources/autosaddler.md) (147 traces vs. 1,400) |
| Execution cost of the resulting artifact | A better score at 19× the tokens may not be better | [HarnessDev](../sources/harnessdev.md), [evolve-the-harness](../sources/evolve-the-harness.md) (token penalty in the score) |
| Regression rate, separately from fix rate | Durability ≠ capability | [AutoSaddler](../sources/autosaddler.md) |
| Evidence the change executed | Guards against dead code | [HarnessDev](../sources/harnessdev.md) trajectory-level component audit |
| Compliance audit against the metric | The metric is an attack surface | [HarnessDev](../sources/harnessdev.md) |

Two sources fold cost directly into the *accept rule* rather than the report, which is stronger: [evolve-the-harness](../sources/evolve-the-harness.md)'s blended `pooled + 0.5·all_pass − 0.005·tokens/M`, and [Squeeze-Evolve](../sources/squeeze-evolve.md)'s Pareto framing (equal-or-better accuracy *at a fraction of cost* — a frontier result, not a point gain).

## Benchmarks Whose Object Is the Loop

[HarnessDev](../sources/harnessdev.md) is the wiki's first ingested member of a fast-growing cluster. Its related-work section names the rest; **none are yet ingested here**, and together they are a ready backlog:

| Benchmark | What it measures |
|-----------|------------------|
| [HarnessDev](../sources/harnessdev.md) | Creation from a weak seed **and** Evolution; creator/executor separated; execution cost; every frozen version scored on a disjoint held-out split |
| Harness-Bench | How harness *choice* changes model performance |
| Meta-Agent Challenge | A meta-agent programs an agent in a sandbox; protected held-out tests across 5 domains |
| HarnessOpt-Bench | Optimizing a *provided* seed harness under a fixed budget; normalized gain on an inaccessible partition |
| Evo-Bench | Evolver models improve a shared CodeAct seed with the runtime model held fixed; sensitivity-calibrated multi-domain suite |
| SEAGym | Intermediate snapshots, cost, and in-/out-of-distribution results — finds later updates need not preserve held-out gains |
| Priority-ranking evaluation | Whether an optimizer identifies the components most *worth* changing |
| "Harness Updating Is Not Harness Benefit" | Separates producing a useful update from an executor's ability to exploit it |

The recurring finding across them — echoed by matched-budget studies that report harness evolution can overfit its search benchmark and may not beat simpler search baselines — is that **held-out durability, not peak feedback-set score, is the discriminating metric.**

## The Honest Reading of This Wiki's Numbers

[HarnessDev](../sources/harnessdev.md)'s held-out gains from a full self-evolution loop are **+1.43 to +4.44 points** (mean +3.11), and **negative for 3 of 4 lineages under a fixed executor**. The wiki's method papers report +9 to +21pp. Both can be true, and the difference is informative rather than embarrassing:

- HarnessDev tests **general-purpose frontier models doing the developer role** with **no method attached**, a small evaluation budget (10 full-eval pairs), and **one trajectory per cell**. It is a measurement of the naive baseline.
- The method papers add exactly the machinery this page is about: dense diagnostic feedback, constrained proposal spaces, mini-batches, reflection, and held-out gates. [AutoSaddler](../sources/autosaddler.md)'s ablations are effectively a demonstration that removing them collapses the gain toward HarnessDev's range (62.0 → 50.6 against a 53.0 base).

Read together: **the gates and the diagnosis are not safety garnish on top of a loop that would work anyway — they are most of what makes the loop work.** And the ceiling for an ungated loop with a noisy visible score is low.

## Open Questions

- HarnessDev holds the **development environment fixed** across both stages and explicitly leaves open whether an evolved harness can serve as the development environment for further evolution. That is precisely the recursive step [STOP](../sources/stop.md) and [Hyperagents](../sources/hyperagents.md) take — so the recursive case is currently *unmeasured* by any benchmark here.
- Is there a **matched-budget** comparison in which harness evolution beats simple random search or best-of-N? HarnessDev flags this as future work, and matched-budget studies it cites suggest the answer is not obviously yes.
- What is the right **noise-floor reporting standard**? [OPHIS](../sources/ophis.md)'s 3σ-over-10-evals is the strictest in the wiki, but it is affordable only because its evaluations are cheap. What is the equivalent discipline when one evaluation costs 89 agent rollouts?
- Agent-built harnesses systematically **declare state and never use it** (11/18 define a `State` class; 1 checkpoints; 0 checkpoint events in 26,679 trajectories) — while [SKILL.state](../sources/skill-state.md) argues explicit execution state is the highest-leverage runtime abstraction available. Why do loops not discover it, and would a taxonomy-constrained proposer like [AutoSaddler](../sources/autosaddler.md)'s find it?
- [WikiSkill](../sources/wikiskill.md) shows that **contaminating the rollout with the diagnostic layer degrades the signal** (Inference-Agent wiki access costs 2.8 points). How many other loops leak their knowledge base into execution without noticing?

## Connections

- [concepts/regression-gating](regression-gating.md) — the mechanisms that enforce what this page measures; reward hacking as the deeper motivation
- [concepts/feedback-signals](feedback-signals.md) — a noisy or offline signal is the root cause of most unreal gains
- [concepts/harness-optimization](harness-optimization.md) — creator/executor separation reframes the model-specificity debate
- [sources/harnessdev](../sources/harnessdev.md) — the anchor source
- [sources/autosaddler](../sources/autosaddler.md) — generalization-aware selection ablated; the fix-rate/regression-rate decomposition
- [sources/wikiskill](../sources/wikiskill.md) — the discovery/execution capability split; programmatic audit trails
- [sources/interaction-trajectory-mining](../sources/interaction-trajectory-mining.md) — the wiki's first explicit negative result, and a worked example of isolating the bottleneck
- [sources/weng-harness-blog](../sources/weng-harness-blog.md) — names "weak evaluators" as the field's leading open challenge

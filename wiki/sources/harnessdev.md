---
title: "HarnessDev: Can LLMs Create and Evolve Their Own Agent Harness?"
type: source
tags: [benchmark, harness-optimization, evaluation, creator-executor-separation, held-out-generalization, transfer, negative-result, dead-code, efficiency]
sources: [harnessdev]
url: https://arxiv.org/abs/2609.01437
code: https://self-developing-agents.github.io/
authors: "Yuhao Wu, Jingyuan Zhang, Jiajun Shi (core); Xinping Lei, Qingshui Gu, Yuxuan Zhang, Zexuan Wang, Chen He, Chen Huang, Maojia Song, Zhiyuan Zeng, Shaowen Wang, Jinkai Liu, Yunfeng Shi, Jiaheng Liu, Shen Yan, Wenhao Huang, Ge Zhang, Wenxuan Zhang"
affiliations: ByteDance Seed, Singapore University of Technology and Design, Georgia Institute of Technology, M-A-P, TokenWave.AI
arxiv: 2609.01437
last_updated: 2026-09-08
---

# HarnessDev (arXiv 2609.01437)

## Summary

HarnessDev is the wiki's first **benchmark for harness development itself**. Every other harness source here reports "our loop improved score X by Y points"; HarnessDev asks the prior question — *how good are frontier models at this job in general, and do the gains survive contact with held-out tasks and a different runtime model?* It shifts the unit of evaluation from task outputs to **runnable infrastructure**: the submitted artifact is a harness, which is then frozen and reused across downstream tasks.

Its answers are sobering, and they function as an external audit of this wiki's optimism:

- Models **can** build working harnesses from a weak seed, but sit well behind mature human-engineered references on **code** and **search/research** (while matching or exceeding them on **writing** and **ML experimentation**).
- Evolution produces gains on the visible feedback set that **shrink by roughly 2–4× on held-out tasks**, and **reverse entirely** under a different runtime model for 3 of 4 lineages.
- Feedback-set score and held-out score move in the same direction only **53.1%** of the time — barely better than a coin flip — and only **2 of 9** creator-declared final versions were actually the held-out-optimal version of their own lineage.

## The Central Design Move: Creator ≠ Executor

```
(L_C, D) → H          # creator LLM works in dev environment D to produce harness H
(H, L_E, x) → y → score   # H is FROZEN; executor LLM L_E runs it on task x; evaluator J scores
```

`D` builds `H`; `L_E` is only used *after* `H` is frozen. This separation is what lets the benchmark distinguish **harness quality** from **executor capability** — a confound that "our harness improved our agent" results cannot separate. Two evaluation modes:

- **Self-Eval** (`L_E = L_C`): measures the complete creator–harness system.
- **Unified-Eval** (fixed `L_E` = Gemini 3.1 Pro): makes harnesses from different creators directly comparable.

The paper cites a concurrent result — *"Harness Updating Is Not Harness Benefit"* — as the motivation, and it is the same distinction [WikiSkill](wikiskill.md) arrives at from the skill side (*discovering* useful procedural knowledge and *executing* it are different capabilities).

## The Weak Seed

Every creator starts from the same `H_seed`: a **runnable compatibility layer, not a task-solving agent**. It parses task and model config, exposes passive low-level tools (paths, files, search, process, LLM gateway, artifact I/O), and writes the required results/trajectories/logs. It may issue one connectivity probe but never attempts the task.

Deliberately absent: agent loop, task decomposition, tool policy, context management, persistent task state, verifier, retry/recovery, stopping rule. **Unmodified, it scores zero on every downstream benchmark** — so any nonzero score is attributable to execution logic the creator added. The creator must implement six control modules:

**E**xecution loop · **T**ools · **C**ontext · **S**tate/memory · **L**ifecycle/recovery · **V**erification

This design threads between two bad options the paper names: an empty repository would confound harness design with CLI/file-format setup; a mature agent would give away the planning and verification structure under test.

## Coverage

**Creation (RQ1)** — four domains, five benchmarks, 2,207 unique downstream instances:

| Domain | Benchmark | Tasks | Metric |
|--------|-----------|-------|--------|
| Code | SWE-bench Pro (public split) | 731 | Task success |
| Code | Terminal-Bench 2.1 | 89 | Task success |
| Data analysis | MLE-bench | 75 | Medal score |
| Writing | EQ-Bench3 | 46 | Rubric score |
| Research | BrowseComp | 1,266 | Accuracy |

**Evolution (RQ2)** — code harnesses only. Feedback set: 100 SWE-Pro tasks + all 89 Terminal-Bench tasks. Budget: **10 post-`H0` full-evaluation pairs**; a candidate enters the official trajectory only after *both* legs complete. Between charged pairs the creator may run at most two 5-task probes (diagnostic, never official). After all trajectories end, every frozen version is additionally scored on a **disjoint 630-instance SWE-Pro held-out split never shown to the creator**.

Six creators: **Opus 4.8, GPT-5.5, Gemini 3.1 Pro, DeepSeek V4 Pro, Qwen 3.7 Max, Seed 2.0 Pro**. Dev environment: Claude Code 2.1.177 (GPT-5.5 uses Codex 0.144.3). Creation reports avg@3.

Both **capability** (held-out task success) and **efficiency** (executor tokens consumed, total and per task) are first-class axes — creator-side build tokens are excluded.

## Creation Results (Self-Eval, avg@3)

| Creator | SWE-Pro | Term-2.1 | MLE-bench | EQ-Bench3 | BrowseComp | Avg |
|---------|---------|----------|-----------|-----------|------------|-----|
| Seed harness | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 |
| **Opus 4.8** | **69.3** | 64.8 | **32.9** | **84.6** | 52.4 | **67.8** |
| Gemini 3.1 Pro | 43.6 | **68.8** | 32.4 | 74.8 | 35.2 | 55.6 |
| GPT-5.5 | 32.8 | 52.1 | 19.1 | 83.0 | 52.6 | 55.1 |
| DeepSeek V4 Pro | 28.9 | 35.6 | 19.6 | 75.4 | 40.9 | 45.2 |
| Qwen 3.7 Max | 33.5 | 41.3 | 3.1 | 68.7 | 32.3 | 44.0 |
| Seed 2.0 Pro | 10.8 | 6.0 | 5.3 | 71.1 | 3.2 | 22.8 |
| *Human reference* | *80.0* | *88.8* | *24.0* | *83.7* | *92.2* | *86.2* |

The domain pattern is the finding: **writing** is matched or beaten (Opus 84.6 vs. 83.7), **ML experimentation** is beaten outright (32.9 / 32.4 vs. 24.0), **code** trails (69.3 vs. 80.0; 64.8 vs. 88.8), and **search/research** is the largest gap (52.4 vs. 92.2 — long-horizon information seeking). Also: **77.8% of failed Data tasks are attributable to harness defects**, not executor capability.

### Artifact pathologies — the most useful part of the paper

Reading the generated harnesses (rather than only their scores) turns up failure modes no score table would show:

- **Edit volume does not predict quality.** The 18 Code artifacts add 17,111 net lines; **Gemini adds the fewest (1,006) yet scores best on Terminal-Bench (68.8)**. Focused changes with frequent verification beat code volume.
- **State and memory is the systematic gap.** 11/18 artifacts define a `State` class, but **only one exposes a state-saving interface and only one implements periodic checkpointing** — and **no checkpoint event appears in 26,679 recorded trajectories.** Execution loops (18/18), verification (15/18), tools and lifecycle (13/18) fare far better.
- **Dead code is pervasive.** Of 108 Code component instances, 72 trigger in real runs, 18 show partial evidence, and **18 are never observed — all of them state/memory**. Beyond Code: **124 of 587 Writing features are confirmed dead**, and 36 Data mechanisms sit on dead paths. Of 169 new functions/classes added during Evolution, 113 are reachable, 31 only through dead code, and **25 have no caller at all.**
- **Validation is mostly syntactic.** 441 of 2,325 executed Data tasks produce degenerate submissions that **no harness detects**.
- **Self-testing only helps when it closes the loop.** Self-test *count* correlates with downstream score at only ρ = 0.13–0.26 (not significant); **revision calls reach ρ = 0.57 (p ≤ .0005)**. Testing pays off when the creator reads the failure, makes a targeted change, and re-verifies.

### Executor transfer: creator co-adaptation is real and large

Portability is a property of the individual harness, not the creator model:

- **Opus's SWE-Pro harness collapses from 69.3 → 33.0** under Gemini; its Writing score drops 84.6 → 74.2. One Opus Code harness **hard-codes a 120-step limit** tuned to its original executor.
- In Opus's Search harness, the **duplicate-query rate rises from 10.1% to 88.2%** when the executor changes — its deduplication, review, and termination rules were adapted to the original model.
- The reverse also happens: **Qwen gains +17.6 on BrowseComp and +12.9 on MLE-bench** under Gemini, meaning its own executor had been the bottleneck.
- Cost is uneven and uninformative: MLE-bench token use varies ~**19×** across creators, and higher cost does not reliably buy a higher score.

## Evolution Results (RQ2)

Nine lineages, 73 official versions, 64 adjacent version switches.

| Runtime | Creator | Feedback pair (H0 → declared) | Held-out-630 (H0 → declared) |
|---------|---------|-------------------------------|------------------------------|
| Self | Gemini 3.1 Pro | 59.9 → 68.7 (**+8.8**) | 48.89 → 51.59 (+2.70) |
| Self | Opus 4.8 | 71.1 → 74.1 (+3.0) | 63.02 → 67.46 (**+4.44**) |
| Self | Qwen 3.7 Max | 41.8 → 55.7 (**+13.9**) | 42.22 → 43.65 (+1.43) |
| Self | DeepSeek V4 Pro | 47.2 → 60.6 (**+13.4**) | 47.30 → 50.48 (+3.17) |
| Self | GPT-5.5 | 59.2 → 65.1 (+5.9) | 48.25 → 52.06 (+3.81) |
| Fixed Gemini | Opus 4.8 | 58.8 → 68.6 (+9.7) | 48.10 → 50.79 (+2.70) |
| Fixed Gemini | Qwen 3.7 Max | 62.1 → 63.2 (+1.1) | 49.52 → 48.41 (**−1.11**) |
| Fixed Gemini | DeepSeek V4 Pro | 47.3 → 53.8 (+6.5) | 43.02 → 40.63 (**−2.38**) |
| Fixed Gemini | GPT-5.5 | 56.6 → 59.1 (+2.4) | 42.22 → 31.90 (**−10.32**) |

All five self-runtime lineages improve on held-out (+1.43 to +4.44, mean **+3.11**) — but the feedback-set gains are 2–4× larger. Under a **fixed** Gemini runtime, **only Opus improves; the other three regress**, GPT-5.5 catastrophically.

### Evolution is not monotonic, and the score is barely readable

Of the 64 official switches: 8 regress on both benchmarks, 16 regress on one, 3 are cross-benchmark trade-offs, 7 produce no measurable change, **27 report gains inside the repeated-run noise band**, **2 have clear positive evidence beyond noise**, and 1 contains no executable code change at all. The **same commit varies by ~±4.75 pair-score points**, so small gains cannot be attributed to code changes from score alone.

**Failure diagnosis is the weakest step of the loop.** The dedicated trajectory-inspection interface is called **twice** across all nine lineages; explicitly inspected cases cover only **0.5%–40.2%** of the 189 feedback tasks. Creators substitute custom scripts and small probes, which disagree with the full evaluation — one GPT-5.5 candidate **passes all five Terminal probes but scores 0.584 on the full set**.

The one clean positive is instructive about what *does* work. Opus notices that **99 of 100 runs report success while only 48 pass**, traces the gap to premature completion, and adds a completion gate. Feedback is useful when it exposes a concrete failure mode and the resulting change is verified end to end. The counterexample from the same set: Qwen's message sanitizer breaks valid Gemini tool-result sequences.

### Constraint compliance: a clean null result

The creator-visible spec prohibits hard-coding instance solutions, deriving patches from task identifiers or filename allowlists, consulting hidden tests/answers/scorer internals, and replacing the provider-neutral runtime interface. Two properties make this checkable rather than advisory: **the score path is isolated from the harness** (a harness's self-reported status is never a scoring input; SWE-Pro credit comes only from the real repository diff, Terminal-Bench credit only from final environment state), and every run retains trajectory/result/metric artifacts alongside the frozen source for post-hoc audit.

Every run in the paper was audited. **No harness obtained score through a prohibited route.** Given the documented [reward hacking](../concepts/regression-gating.md) in [STOP](stop.md) and [DGM](dgm.md), this is a meaningful negative — and it is a demonstration that *making the score unassertable by the agent* is an effective structural defense.

## What Is Optimized / Feedback / Gating

| Axis | HarnessDev |
|------|------------|
| Optimization target | The harness itself (E/T/C/S/L/V) — but as the *object of evaluation*, not of a proposed method |
| Feedback signal | Creation: 1–3 dev cases. Evolution: full scores on a designated 189-task feedback set + ≤2 five-task probes |
| Loop structure | Two staged single-creator loops: build-from-seed, then revise-own-artifact |
| Gating | Creator-declared final version (self-selected); the benchmark then evaluates *every* frozen version on a disjoint held-out split it never showed the creator |
| Human involvement | Benchmark construction, constraint spec, and post-hoc audit; the development loops themselves are autonomous |

## Why It Matters

- **It supplies the control the rest of the literature lacks.** Held-out gains of +1.43 to +4.44 points from a full self-evolution loop are an order of magnitude below the +9 to +21pp headline numbers reported by [AutoSaddler](autosaddler.md), [Self-Harness](self-harness.md), [HALO](halo.md), and [evolve-the-harness](evolve-the-harness.md). Those systems are not thereby refuted — they use denser feedback, larger budgets, and purpose-built gates, and HarnessDev deliberately tests *general-purpose frontier models doing the developer role* with a small evaluation budget and no method attached. But it fixes the ceiling for the naive version and shows what the gates are buying.
- **"Visible feedback is useful for local search but unreliable for final selection."** The 53.1% direction-agreement rate and 2/9 held-out-optimal declarations are the crispest statement in the wiki of why [held-out gating](../concepts/regression-gating.md) is not optional: repeatedly optimizing a noisy score favors a lucky run and amplifies overfitting.
- **Creator co-adaptation is a measurable failure mode, not a worry.** Opus's 69.3 → 33.0 collapse and the 10.1% → 88.2% duplicate-query blowup give mechanism to the [model-specificity debate](../concepts/harness-optimization.md). It sides with [Self-Harness](self-harness.md) on *how much* harnesses co-adapt, while showing the co-adaptation is often an accident (a hard-coded step limit) rather than a genuine per-model optimum.
- **State/memory is where agent-built harnesses are weakest.** No checkpoint event in 26,679 trajectories, and every never-triggered component concerning state, is a specific and actionable gap — and a striking counterpoint to [SKILL.state](skill-state.md), which argues that explicit execution state is the single highest-leverage runtime abstraction. Models apparently know to *declare* state and not to *use* it.
- **Efficiency belongs in the objective.** 19× token spread with no reliable relationship to score argues that harness quality reports without cost numbers are incomplete — the same case [evolve-the-harness](evolve-the-harness.md) makes by folding a token penalty into its promotion score.

## The Benchmark Cluster Around It

HarnessDev's related-work section reveals that "evaluating harness self-improvement" became a subfield in 2026. Named siblings, none yet ingested here:

| Benchmark | Focus |
|-----------|-------|
| Harness-Bench | How harness *choice* changes model performance |
| Meta-Agent Challenge | Meta-agent programs an agent in a sandbox; protected held-out tests, 5 domains |
| HarnessOpt-Bench | Optimizing a *provided* seed harness under a fixed budget; scored by normalized gain on an inaccessible partition |
| Evo-Bench | Evolver models improve a shared CodeAct seed with the runtime model held fixed |
| SEAGym | Records intermediate snapshots, cost, and in-/out-of-distribution results |
| Priority-ranking evaluation | Whether an optimizer identifies the components most worth changing |
| "Harness Updating Is Not Harness Benefit" | Separates producing a useful update from an executor's ability to exploit it |

It also names harness-optimization methods the wiki hasn't covered: **VeRO** (versioned snapshots, budget-controlled evaluation), **HarnessFix** (provenance/control-flow-aware localized repair), **DemoEvolve** (demonstrations when reward feedback is unreliable), **HarnessCompass** (overfitting and component interference), **Harness-R1** (trains a harness engineer to convert failure batches into validated patches), and **HarnessX** / **Co-Harness** (composable harness foundry; co-evolving harness and weights). A ready ingest backlog, in the same way [Weng's survey](weng-harness-blog.md) was.

## Limitations (the paper's own)

Four categories don't cover all deployments; human baselines are uneven and not guaranteed optimal (and are not paired controls under a common executor — the "% of human reference" figure measures distance to *selected mature systems*, not to human ability). Unified-Eval reduces but cannot remove executor-model differences. Evolution has **one trajectory per creator–runtime cell**, so no uncertainty estimates or population-level comparisons; its held-out evaluation covers SWE-Pro only. Matched-budget comparison against simpler search baselines is left as future work.

Most interesting for this wiki: **the development environment `D` is held fixed across both stages** — whether an evolved harness can itself serve as the development environment for further evolution is explicitly left open. That is precisely the recursive step [STOP](stop.md) and [Hyperagents](hyperagents.md) take, and HarnessDev declines to measure it.

## Connections

- [concepts/evaluating-self-improvement](../concepts/evaluating-self-improvement.md) — the anchor source for this concept
- [concepts/harness-optimization](../concepts/harness-optimization.md) — supplies the creator/executor separation and the co-adaptation evidence
- [concepts/regression-gating](../concepts/regression-gating.md) — feedback-vs-held-out divergence; unassertable scores as a structural anti-hacking defense
- [concepts/feedback-signals](../concepts/feedback-signals.md) — diagnosis as the weakest link; probes that disagree with full evaluation
- [concepts/knowledge-accumulation](../concepts/knowledge-accumulation.md) — state/memory as the systematic gap in agent-built harnesses
- [sources/autosaddler](autosaddler.md) — the method-side counterpart, with far larger gains under much stronger gating
- [sources/skill-state](skill-state.md) — argues explicit execution state is the key abstraction; HarnessDev shows models won't build it unprompted
- [sources/self-harness](self-harness.md), [sources/evolve-the-harness](evolve-the-harness.md) — the model-specificity / transfer debate this source informs
- [sources/meta-harness](meta-harness.md) — cited as a closely related method (coding-agent proposer over prior candidates/scores/traces)
- [sources/weng-harness-blog](weng-harness-blog.md) — its "weak evaluators" challenge is what HarnessDev attacks

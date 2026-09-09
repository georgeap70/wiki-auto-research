---
title: "AutoSaddler: Automatic Harness Optimization with Durable Updates from Agent Execution Traces"
type: source
tags: [harness-optimization, offline-learning, mini-batch, patch-taxonomy, structured-intervention, generalization-aware-selection, evodag, regression-gating, feedback-signals]
sources: [auto.saddler]
url: https://arxiv.org/abs/2608.23041
code: https://aka.ms/AutoSaddler-website
authors: Sungho Park, Wonjoong Kim, Rongyuan Tan, Jue Zhang, Wook-Shin Han, Pengfei Gao, Chanyoung Park, Yongqiang Yao, Rao Fu, Elsie Nallipogu, Qingwei Lin, Saravan Rajmohan, Dongmei Zhang
affiliations: POSTECH, KAIST, Southern University of Science and Technology, Microsoft
arxiv: 2608.23041
last_updated: 2026-09-08
---

# AutoSaddler (arXiv 2608.23041)

## Summary

AutoSaddler recasts [harness optimization](../concepts/harness-optimization.md) as an **offline supervised-learning problem** and runs it with the machinery of mini-batch SGD: batches of training tasks, a diagnosis step that plays the role of backpropagation, a patch taxonomy that plays the role of a constrained parameter space, a phased schedule that plays the role of a learning-rate schedule, and a dev-set gate that plays the role of early stopping. (The name is a pun — a saddler is a maker of harnesses.)

Its central empirical claim is a decomposition: effective harness optimization needs **three separable ingredients**, and the paper ablates each one.

| Ingredient | What it replaces | GAIA2 Pass@1 when removed |
|------------|------------------|---------------------------|
| **In-depth diagnosis** | Shallow single-call reflection | 62.0 → 57.8 |
| **Structured intervention** | Unconstrained editing | 62.0 → 56.9 |
| **Generalization-aware selection** | Trajectory-specific repair | 62.0 → **50.6** |

The largest drop is generalization-aware selection — the paper's strongest result is that *selecting for durability matters more than proposal quality*.

## The Optimization Problem

The harness is factored into three parameter blocks:

```
θ = (θ_prompt, θ_tool, θ_middleware)
```

- `θ_prompt` — instructions and system prompts
- `θ_tool` — available tools and their interfaces
- `θ_middleware` — runtime control logic: hooks, agent-loop behavior

Memory and skill curation are explicitly **out of scope** (the paper assumes stateless, independent tasks) — a deliberate contrast with [WikiSkill](wikiskill.md) and [SkillOpt](skillopt.md), which optimize exactly that layer. The objective `J(θ) = E[µ(ŷ, y*)]` is maximized under a **rollout budget K**, with the final answer chosen as the best dev-set scorer among candidates evaluated within budget.

## The Loop: Three Sessions + EvoDAG

Each iteration `n` runs the current harness `H_n` on a mini-batch `B_n ⊂ D_train`, then:

**1. Diagnosis–Patch Session.** Diagnosis and patch generation are deliberately **not separated**, so the agent can use context gathered while investigating. The agent gets the failure traces *and the harness codebase*, plus structured guidance for progressively retrieving trace detail (mitigating the long-context problem that limits single-trace diagnosers). It identifies a suspected root cause, considers alternative hypotheses, then emits a structured patch `Δθ_n`. Only files implementing the harness's functional logic are exposed; evaluation and benchmark-data code is off-limits — a [leak guard](../concepts/regression-gating.md) in the same spirit as [evolve-the-harness](evolve-the-harness.md)'s `_touched_test()`.

**2. Verify.** The patch is a *mini-batch improvement* if `Ĵ_Bn(H'_n) > Ĵ_Bn(H_n)`. Only then does it earn a (more expensive) dev-set evaluation.

**3. Reflection Session.** Pre- and post-patch traces are compared and every task sorted into **fixed / regressed / still-failing / still-passing**, each with targeted self-reflection questions (why did this work, why did that regress, why was the patch insufficient, did it do anything at all). If a dev-set run happened, reflection additionally addresses whether the patch *generalizes* beyond the mini-batch.

**4. Evolution Session.** Lessons, patch descriptions, and scores are stored as node attributes in **EvoDAG**, a directed acyclic graph where nodes are explored harnesses and edges are diffs. The Evolution Agent consults the *whole* DAG to synthesize `H_{n+1}` — it may **recombine components from any subset of prior harnesses**, not just continue from `H'_n`. This is the escape-local-optima mechanism, and it makes AutoSaddler a hybrid: hill-climbing within an iteration, LLM-driven evolutionary recombination across them.

All three agents are implemented on the **Claude Agent SDK**; the optimizer LLM is Claude Opus 4.6.

## Structured Intervention: The Patch Taxonomy

The patch space is enumerated rather than left open, and each subtype is tagged **Capability (C)** — changes executable code or orchestration — or **Steering (S)** — textual edits leaving code unchanged.

| Category | Subtype | C/S |
|----------|---------|-----|
| Prompt | Prompt Rule Addition | S |
| Prompt | Prompt Rule Modification | S |
| Tool | New Tool Addition | C |
| Tool | Argument Modification | C |
| Tool | Implementation Fix | C |
| Tool | Tool Description Fix | S |
| Middleware | PreToolUse Hook | S |
| Middleware | Infrastructure Change | C |
| Middleware | Agent Loop Logic Change | C |

**Phased Patch Scheduling** — explicitly analogized to learning-rate scheduling — begins in a Capability phase and transitions to a Steering phase. Removing *only* the schedule costs 60.7 → 54.8 Pass@1; removing the whole taxonomy as well costs another 1.5 points.

### Why the taxonomy earns its keep

Without structured intervention, patch generation **collapses onto Steering edits (91.5%)** — cheap prose tweaks — while barely touching tools or infrastructure. AutoSaddler's schedule produces 65.8% Capability / 34.2% Steering. This matters because the highest-acceptance patch types are precisely the capability-centric ones:

| Subtype | Acceptance rate |
|---------|-----------------|
| New Tool | 83% |
| Loop Change | 71% |
| Infra Change | 67% |

Under the unconstrained ablation these three account for only **4%** of generated patches; AutoSaddler raises their share above 25%. And Capability patches are **more durable**: comparable fix rate to Steering (55% vs. 58%) but less than half the regressions (8% vs. 17%).

This is independent, mechanism-level confirmation of [evolve-the-harness](evolve-the-harness.md)'s headline finding that **deterministic code beats prompts** — and it adds a causal explanation for why prompt-only optimizers underperform: left unconstrained, an LLM optimizer *prefers* prose edits, because they are easier to write, not because they work better.

## Results

Test-set Pass@1, mean ± SD over three runs, with train/dev/test drawn from **disjoint task groups** (e.g. SWE-Bench Pro: train = qutebrowser, dev = Vuls + NodeBB, test = Ansible + Flipt + Element-web), so the numbers measure cross-repository generalization.

| Benchmark | Base harness | GEPA | Meta-Harness | AutoSaddler |
|-----------|--------------|------|--------------|-------------|
| GAIA2 | 53.0 (default ReAct agent) | 54.6 | 53.2 | **62.0** (+9.0) |
| SWE-Bench Pro | 37.3 ([SWE-agent](https://github.com/SWE-agent/SWE-agent)) | 42.5 | 35.3 | **46.9** (+9.6) |
| Terminal-Bench 2.0 | 40.0 (Terminus 2) | 42.5 | 43.3 | **50.0** (+10.0) |

Gains over the strongest *automated* baseline per benchmark: **+7.4 / +4.4 / +6.7**. On TB2 it also beats **Terminus KIRA**, a manually expert-tuned harness, 50.0 vs. 47.5.

*(Note: §5.2 of the paper misstates two of these deltas — "+8.4 pp" on SBP and a swapped SBP/TB2 baseline gap. The table values and the conclusion agree; the numbers above are computed from the tables.)*

### Efficiency — the sharpest practical result

| Measure | AutoSaddler | GEPA | Meta-Harness |
|---------|-------------|------|--------------|
| Peak dev accuracy (GAIA2) | **72.3%** @ ~1,000 rollouts | 64.6% @ ~2,800 | 61.5% @ ~2,800 |
| Traces consumed to reach best dev score | **147** | — | 1,400 (**~10×** more) |

With selective evaluation it reaches 67.7% dev accuracy on 391 rollouts — already above Meta-Harness's 1,400-rollout peak. Optimizer-side dollar cost per patch is moderately higher, but rollouts (the dominant cost) drop by an order of magnitude.

### Robustness and transfer

- Independent re-run on GAIA2: **58.6%** — still well above base and reruns of baselines.
- Optimization on a *different* training universe: **57.4%** (+5.9 over base).
- **Cross-model**: swapping the task agent's LLM from Opus 4.6 to **Haiku 4.5** while keeping the Opus-optimized harness still yields **+5.6pp** over base — partial transfer, consistent with [evolve-the-harness](evolve-the-harness.md) and against [Self-Harness](self-harness.md)'s strict model-specificity claim.

## Generalization-Aware Selection in Detail

The RQ3 ablation is the most instructive part of the paper. Removing reflection + dev-set evaluation leaves fix rate **essentially unchanged** — both settings find equally good repairs — but splits them on the **regression rate**:

- AutoSaddler: regression trend **−0.24 pp/iter** (decreasing)
- Ablation: regression trend **+0.16 pp/iter** (increasing)

A concrete case: at iteration 20 the ablation adds a `send_progress_message_to_user` tool and rewires the hook on the high-frequency `send_message_to_user` tool to force redirection to it. With no reflection to assess collateral damage, the over-scoped patch is retained and the dev regression rate jumps 8% → 22%. **The identical failure pattern — an over-broad hook on a high-frequency tool — also appears in AutoSaddler at iteration 4, and reflection blocks it.**

The lesson: proposal quality and durability are different problems. A harness optimizer without a generalization gate will keep finding real fixes and keep shipping them alongside real regressions.

## What Is Optimized / Feedback / Gating

| Axis | AutoSaddler |
|------|-------------|
| Optimization target | Harness code: prompts + tools + middleware (no memory/skills, no weights) |
| Feedback signal | Failure traces + harness source, explored agentically (deep debugging, not single-call reflection) |
| Loop structure | Offline mini-batch hill-climb with LLM-driven recombination over an EvoDAG archive |
| Gating | Three-stage: mini-batch improvement → dev-set generalization check → reflection-informed retention; plus prior restraint via patch taxonomy + phased schedule |
| Human involvement | None after setup (splits, base harness, budget) |

## Why It Matters

- **The mini-batch/SGD analogy made load-bearing.** [SkillOpt](skillopt.md) framed skill-document optimization as gradient descent on prose; AutoSaddler carries the analogy further into harness *code* and adds the pieces SkillOpt's framing lacked: mini-batches, an explicit train/dev/test split across task groups, and a learning-rate-like phase schedule. Diagnosis-patch-verify is positioned as textual backpropagation *with mandatory empirical checking*, on the grounds that textual "gradients" — unlike numerical ones — are unverified hypotheses.
- **Prior restraint beats post-hoc filtering for search-space control.** The taxonomy + schedule are a [prior-restraint gate](../concepts/regression-gating.md) in the same family as SkillOpt's edit budget and [OPHIS](ophis.md)'s plausibility filter, but restraining *category* rather than size or semantics. Its effect is to force exploration away from the LLM's cheap-prose attractor.
- **Direct comparison against two wiki systems.** AutoSaddler is the first wiki source to benchmark [GEPA](optimize-anything.md) and [Meta-Harness](meta-harness.md) head-to-head on the same three benchmarks, and it beats both on all three. It attributes Meta-Harness's shortfall specifically to *unconstrained editing* — the same ablation as its own "w/o structured intervention" arm.
- **Explicitly not self-referential.** The paper declines the [DGM](dgm.md)/[Hyperagents](hyperagents.md) framing (meta-agent = task agent), arguing that under a finite rollout budget the goal is improving the task agent's harness, not modeling self-referential dynamics.

## Connections

- [concepts/harness-optimization](../concepts/harness-optimization.md) — a mini-batch offline-learning formulation with a typed patch space
- [concepts/self-improvement-loop](../concepts/self-improvement-loop.md) — three-session loop with an EvoDAG archive; recombination across lineages
- [concepts/feedback-signals](../concepts/feedback-signals.md) — "deep debugging vs. shallow reflection", quantified (6.2 extra tool calls, 5.8 extra file accesses per step; 13 vs. 5 accepted patches by end of epoch 1)
- [concepts/regression-gating](../concepts/regression-gating.md) — generalization-aware selection; the fix-rate/regression-rate decomposition
- [concepts/knowledge-accumulation](../concepts/knowledge-accumulation.md) — EvoDAG as a lessons-annotated lineage graph
- [concepts/evaluating-self-improvement](../concepts/evaluating-self-improvement.md) — disjoint-task-group splits as the generalization protocol
- [sources/meta-harness](meta-harness.md), [sources/optimize-anything](optimize-anything.md) — the two baselines it beats
- [sources/evolve-the-harness](evolve-the-harness.md) — converges on code-over-prompts, from a different direction
- [sources/harnessdev](harnessdev.md) — the benchmark-side counterpart; much less optimistic about evolution's durability
- [sources/self-harness](self-harness.md) — contrast: strict model-specificity vs. AutoSaddler's partial cross-model transfer

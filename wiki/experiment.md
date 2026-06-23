---
title: Experiment — Optimizing Vulnerability-Detection Scanning Workflows
type: analysis
tags: [experiment, vulnerability-detection, security, claude-code-skills, harness-optimization, application]
sources: [skillopt, evo-hq, honedhaiku, optimize-anything, rlm_gepa, auto-harness]
last_updated: 2026-06-23
---

# Experiment — Optimizing Vulnerability-Detection Scanning Workflows

## Use case

Investigate the effectiveness of various **scanning workflows for vulnerability detection in source code**, where each workflow is implemented as a **Claude Code skill** and evaluated against a set of source repositories with **ground-truth vulnerability labels**.

The setup is unusually well-fit to the harness-optimization literature in this wiki:

| Property | Implication |
|----------|------------|
| Artifact being optimized | Claude Code skills — i.e., text-level harness artifacts (markdown skill docs, tool wiring, prompts) |
| Feedback channel | Ground-truth labels → both a scalar (precision/recall/F1) and rich per-failure descriptions (which CVE was missed, which file produced a false positive, which CWE category degraded) |
| Search space | Discrete prose + small code edits to the skill |
| Generalization risk | High — overfitting to specific repos or specific CWEs is the central failure mode |
| Model | A **categorical optimization axis** spanning closed-weight (Claude) and open-weight models — selected among, not weight-updated (see [multi-objective extension](#multi-objective-extension--optimizing-across-precision-recall-cost-model) below) |
| Objectives | **Multi-objective**: precision, recall, **and** per-scan runtime cost — in genuine tension, so a Pareto frontier rather than a single scalar |

Because no model is weight-updated (open-weight models are *selected among*, not fine-tuned) and the artifact is a text-level skill, the entire wiki section on **weight optimization** ([sources/agentflow](sources/agentflow.md), [sources/trace](sources/trace.md), [sources/skill-rl-skill0](sources/skill-rl-skill0.md)) is out of scope from the start. The relevant literature is **harness optimization** + **text-artifact optimization**. (If you ever decide to *train* the open-weight models rather than select among them, that cluster re-enters scope.)

## Primary recommendation

### [sources/skillopt](sources/skillopt.md) (Microsoft Research) + [sources/evo](sources/evo.md) (evo-hq), combined

Use **Evo** as the orchestration substrate and **SkillOpt's discipline** as the proposal style inside it.

#### Why SkillOpt

It is the only system in the wiki that *literally* treats a skill document (`best_skill.md`) as the trainable state of a frozen LLM. The mechanism matches your artifact exactly:

- **Bounded add/delete/replace edits** ("textual learning rate") prevents the optimizer from rewriting a working skill to chase a marginal failure — important when vuln-detection knowledge accumulates across many CWE categories
- **Held-out validation gate** on every edit (matches your need for a held-out repo split)
- **Rejected-edit buffer** as a persistent negative-signal store — when an edit fails the gate, it is retained as structured "what not to try" rather than discarded; the optimizer reads from this buffer on future rounds
- **Best-or-tied-best on 52/52 settings** across 7 models × 6 benchmarks, with the strongest cross-model (+15.2%) and cross-harness (+31.8%) transfer evidence in the wiki — directly relevant if you want the optimized skill to keep working when you change the underlying model (Sonnet → Opus, or vice versa)

The "textual learning rate" framing is the load-bearing intuition: **a smaller edit budget widens the productive band** ([concepts/regression-gating](concepts/regression-gating.md)). Vuln detection is a domain where you want the optimizer to *refine* a careful checklist of patterns, not regenerate the prompt from scratch every round.

#### Why Evo

Of all the orchestrators in the wiki, Evo is the only one that **ships as Claude Code skills** (`/evo:discover`, `/evo:optimize`) and is structurally tailored to "optimize a skill against a benchmark in a repo." The map onto your problem:

| Your setup | Evo primitive |
|------------|---------------|
| "Define what to scan and how to score it" | `/evo:discover` — interactive setup of benchmark + metric direction; can be seeded with a one-line directive |
| "Don't overfit to my labeled repos" | **Held-out-slice score-floor gate auto-attached at discovery** — generalization protection is the default, not something you have to remember to wire up |
| "Run experiments in parallel" | Subagents in isolated git worktrees |
| "Maintain specialists across CWE categories" | `pareto_per_task` frontier strategy — credited to [GEPA](sources/optimize-anything.md); keeps branches that win on specific CWEs even when their aggregate lags |
| "Don't accept a regression even if F1 went up" | **Gates inherit down the experiment tree and hard-veto over score** — *"An experiment that fails a gate is discarded even if its score beats the current best"* |
| "Find compound failure patterns across runs" | RLM-inspired **cross-cutting scan subagents** between rounds — surface gate-failure intersections and shared root causes across traces |
| "Scans are slow; I want overnight runs" | Eight execution backends (worktree/pool/ssh/modal/e2b/daytona/aws/azure) |
| "Inspect what the experiments did" | Dashboard with frontier strategy configuration |

The `pareto_per_task` strategy is particularly load-bearing for vulnerability detection: precision and recall are in genuine tension, and CWE categories have different optimal patterns. Collapsing everything into one F1 scalar will lose specialist branches that detect XSS well even if they regress on SSRF. Pareto-per-task keeps them as live parents for further exploration.

#### How to compose them

| Layer | Use | Role |
|-------|-----|------|
| Orchestration | Evo | Experiment tree, gating, frontier selection, multi-backend execution, dashboard |
| Proposal style | SkillOpt-style | Bounded edits per round; separate success/failure reflection; maintain a rejected-edit buffer alongside Evo's discarded-hypothesis store |
| Gating | Evo's gates + SkillOpt's held-out | Held-out repo gate (auto-attached by `discover`) + per-CWE score-floor gates attached to relevant branches |
| Generalization probe | SkillOpt's cross-model transfer | Periodically score the current best skill under a *different* Claude model to detect overfitting |

## Multi-objective extension — optimizing across (precision, recall, cost, model)

The framing above optimizes detection quality with the model held fixed. The real objective is broader: **maximize precision and recall while minimizing per-scan runtime cost, across a discrete set of closed- and open-weight models.** This changes the problem in two specific ways that reshuffle the method ranking.

### The two structural shifts

1. **Cost makes a Pareto frontier mandatory.** With (precision, recall) alone you could half-justify collapsing to F1. Adding cost gives three numeric objectives in genuine tension — a slower, more thorough scan buys recall; a cheaper model buys cost at the expense of precision. Any method that gates or selects on a *single scalar* now actively destroys the thing you care about: the tradeoff curve. This **promotes [GEPA](sources/optimize-anything.md) from "underlying primitive / DIY fallback" to the core selection mechanism**, and **demotes scalar-gated methods** ([auto-harness](sources/auto-harness.md)'s 80% regression gate, plain F1-driven [HonedHaiku](sources/honedhaiku.md)) to single-objective control arms only.
2. **Model becomes a search axis, not a frozen base.** The original framing treated cross-model behavior only as a *generalization probe*. Here `model` is a first-class categorical coordinate spanning open + closed weights.

### Does GEPA handle this out of the box?

Split the answer by axis — GEPA treats the numeric objectives and the model axis very differently.

**(precision, recall, cost) — yes, this is GEPA's advertised sweet spot, but with real plumbing.** GEPA's own design names these exact objectives: *"Rather than collapsing multiple objectives (correctness, speed, cost, robustness) into a weighted scalar, GEPA maintains a Pareto frontier of non-dominated candidates"* ([optimize-anything](sources/optimize-anything.md)). You will **not** need to extend the core algorithm. Three things are on you:

- **You write the cost metric — GEPA does not measure it.** GEPA's contract is `optimize(artifact, evaluate, side_info)`; cost is whatever your `evaluate` returns. Instrument `$/scan` = (input+output tokens × per-model price) + wall-clock if latency matters. This is the bulk of the work.
- **GEPA *searches* on instance-Pareto + a scalar, but *tracks* an objective-Pareto for free** — resolved by reading the `gepa-ai/gepa` source (`core/adapter.py`, `core/state.py`, verified 2026-06-23):
  - `EvaluationBatch.scores` is typed `list[float]` — **one scalar per evaluation instance**. GEPA sums these for minibatch acceptance, averages them over the valset, and `state.py`'s `_update_pareto_front_for_val_id` builds the frontier **per validation instance** (a candidate joins iff it is best on at least one instance). This is instance-Pareto — a *diversity* mechanism, **not** dominance over `(P, R, cost)`.
  - There **is** a separate `objective_scores` field (optional per-example `{objective_name → score}` map) and an `_update_objective_pareto_front` method — but **no selection or frontier-membership logic consumes it**; it is aggregated and stored for reporting only.
  - **Consequence — no need to patch GEPA's frontier code.** The recipe is: (1) feed `scores` a single per-instance **scalarization** that drives the search (e.g. F-β, or F1 penalized by a cost term); (2) feed the raw `{precision, recall, cost}` into `objective_scores` so GEPA maintains and records the true objective frontier; (3) at the end, read out that objective frontier (or run your own non-dominated filter over `prog_candidate_objective_scores`) and pick your operating point there.
  - **The real limitation is exploration bias, not missing machinery.** Because search pressure comes from the scalarized score, GEPA may under-explore frontier regions your scalarization disfavors (weight recall heavily and it may never select a low-cost/high-precision candidate for mutation, even though that candidate belongs on the objective frontier). **Mitigation:** vary the scalarization across runs (sweep the cost weight / β) and **union the objective frontiers** — which is exactly what Evo's outer per-model sweep + multiple runs gives you. This is the load-bearing reason to keep Evo on top rather than running bare GEPA.
- **Wire ASI for cost too.** For P/R, evidence-bounded feedback is "missed CVE-X at file Y." For cost it is "spent $0.40/scan because it re-reads every file 3×" — otherwise the proposer cannot reason about *why* a candidate is expensive and will only thrash on it.

**model — no, GEPA cannot do this axis.** GEPA optimizes *text artifacts*; a model identifier is a categorical config, not text GEPA mutates. Two options:

- **Sweep it outside GEPA (recommended).** Because the model range is small and discrete, run the skill optimizer once per model, then pool every `(skill, model)` candidate into **one combined `(P, R, cost)` Pareto frontier**. The model axis lives in the orchestration layer ([Evo](sources/evo.md)) — different branches/backends use different models, and Evo's non-Claude backends (modal/e2b/daytona/aws) are what let you run the open-weight models at all.
- **Smuggle it into the artifact.** Put a `model:` directive *inside* the text skill so a proposal can flip it. GEPA then "optimizes over model" but is blind to the cost/capability structure of the choice — fragile; only worth it if model interacts with the prose in non-obvious ways. With a handful of models, prefer the sweep.

### SkillOpt's role changes: from co-optimizer to enabler of the model axis

[SkillOpt](sources/skillopt.md)'s headline property — strongest cross-model transfer in the wiki (+15.2%) — stops being a nice-to-have generalization probe and becomes **the property that makes the model axis exploitable**: a skill optimized to transfer can be deployed on a *cheaper* open-weight model to hit the cost objective without falling off a quality cliff. Use SkillOpt-style bounded edits to keep skills model-robust, then evaluate each `(skill, model)` combination on the combined frontier.

### The two-Pareto subtlety

Both Evo's `pareto_per_task` and GEPA's instance-Pareto are **Pareto-over-tasks** (keep CWE specialists), *not* **Pareto-over-objectives** (precision vs. recall vs. cost). You need **both** frontiers:

- **per-task** — CWE specialists (already in the plan via `pareto_per_task`)
- **per-objective** — the `(P, R, $/scan)` surface (you must add this explicitly; it is not the same knob)

Score each candidate as a vector `(macro-P, macro-R, $/scan)`, keep the non-dominated set, and let `model` be a coordinate of each frontier point — the deliverable is a frontier you pick an operating point on per deployment budget, not a single "best skill."

### Methods worth reconsidering under the wider objective

- **[AutoReason](sources/autoreason.md) becomes a cost lever, not a free add-on.** Its per-finding tournament improves precision but *multiplies inference cost per scan* — it moves you along the cost axis. Treat it as a tunable knob inside the cost objective.
- **[group-evolve](sources/group-evolve.md) and [evoforge](sources/evoforge.md)** — previously skipped as overkill — are genuine multi-objective shapes (group-evolve's performance-*novelty* selection especially). Still heavier than Evo+GEPA, but if model+cost widens the search space well beyond skill-prose-only, they are less of an overreach than the skip table claims. [EvoX](sources/evox.md) (meta-evolution of strategy) stays overkill.

### Net recommendation

Keep Evo as the substrate. Use **GEPA as the per-model skill optimizer**, driving its search with a **cost-aware scalarization** in `scores` while logging raw `{precision, recall, cost}` to `objective_scores` so GEPA records the objective frontier (confirmed available — see above). Add an explicit **cost gate**; sweep **model as an outer categorical**; **union the recorded objective frontiers** across models and across scalarization weights into one `(P, R, cost)` frontier, and pick the operating point there. Use **SkillOpt's cross-model transfer** as the property that makes the model axis safe.

The earlier open question is now **resolved**: GEPA's shipped `scores` is `list[float]` (per-instance scalar), so it does not *search* an objective Pareto — but it *tracks* one via `objective_scores`, so no frontier-code patch is needed. The remaining work is designing the scalarization and unioning frontiers across runs, not extending GEPA's core.

## Secondary fits — useful as control arms or supplements

### [sources/honedhaiku](sources/honedhaiku.md) (Tim Waldin) — closest empirical analog

HonedHaiku applies GEPA to Claude Haiku system prompts for **bug fixing** — the security-shaped sibling of vulnerability detection. It reports +19.7pp on unseen bugs (Haiku 3.5: 65% → 85%) and converged in 4 of 20 allocated iterations.

Two lessons port directly:

- **The Goldilocks band**: prompt-only optimization moved the needle in the ~50–70% baseline range. Below 50%, the model couldn't execute complex methodologies; above ~85%, the prompt was no longer the bottleneck. **Check your baseline detection rate first**: if it's outside this band, expect smaller returns from prompt-only optimization — and lean on SkillOpt's bounded edits, which partially widen the band.
- **Training diversity matters more than iteration count**: a 3-challenge run overfit; 20 challenges across 5 repos generalized. The analog: don't optimize against 3 hand-picked CVEs; sample broadly across CWE categories and repo styles.

### [sources/optimize-anything](sources/optimize-anything.md) (GEPA) — the underlying primitive

GEPA is the search primitive both HonedHaiku and the suggested `pareto_per_task` Evo strategy rest on. If you want to *implement* the optimizer yourself rather than use Evo's packaged form, GEPA is the reference. The core idea — Pareto frontier over multi-metric scoring + LLM-proposed edits + **Actionable Side Information (ASI)** as the feedback channel — is exactly the shape of your problem.

### [sources/rlm-gepa](sources/rlm-gepa.md)'s `AgentSpec` — adopt the *idea* even if not the framework

Even if you don't use the predict-rlm runtime, write down explicitly what behavioral changes are *in-scope* for the optimizer. Example:

```
AgentSpec for the vuln-scanning experiment:

Use cases:
  - CWE-79 (XSS), CWE-89 (SQLi), CWE-22 (path traversal), CWE-78 (command injection)
  - PHP, JavaScript/TypeScript, Python backends

Runtime affordances:
  - Tools: ripgrep, ast-grep, semgrep CLI, file-read, file-list
  - No internet access; no LLM-as-judge inside the skill

Scoring methodology:
  - Per-CWE precision/recall on held-out repo split
  - Aggregate: F1 macro across CWE categories (NOT micro)

Counterfactual axes for generalization:
  - Repos not in the train split
  - CWE categories not seen during optimization (held-out CWE eval)
  - Different Claude model than the one used during optimization

Out-of-scope:
  - Expanding to new languages
  - Adding runtime monitoring or fuzzing
  - Modifying tool definitions (the tools are fixed)
```

Vulnerability detection is exactly the domain where an optimizer drifts if scope isn't declared — it will start adding language-detection heuristics, rewriting tool wiring, etc. AgentSpec is cheap insurance.

### [sources/auto-harness](sources/auto-harness.md) (NeoSigma) — single-thread baseline

The closest single-thread control arm: agent edits its own harness, gates with an 80% regression threshold, keeps a `learnings.md` between sessions. If you want a *baseline* that does not depend on Evo's tree-search or SkillOpt's bounded edits, this is the simplest reproducible loop in the wiki.

## What about scan-time refinement?

### [sources/autoreason](sources/autoreason.md) — orthogonal, not core, but worth considering at scan time

AutoReason's per-query tournament (incumbent vs. adversarial revision vs. synthesis, with a blind Borda-count judge panel) is an **inference-time** loop. It does not optimize the skill. But it could resolve a different problem inside your scanner: *"Is this candidate finding actually a vulnerability or a false positive?"* The tournament gating addresses three pathologies that show up in naive critique-revise (prompt bias, scope creep, lack-of-restraint) — all of which are real in security triage.

Use it as a **separate axis** in your workflow comparison, not as an optimizer choice.

## What to skip and why

| Skipped | Why |
|---------|-----|
| [sources/agent0](sources/agent0.md) | Two-agent bootstrap from zero data — you have ground truth, this isn't the right tool |
| [sources/agentflow](sources/agentflow.md) | RL on planner weights — Claude is closed-weight |
| [sources/trace](sources/trace.md) | LoRA adapters per capability — same closed-weight problem |
| [sources/skill-rl-skill0](sources/skill-rl-skill0.md) | RL training; closed-weight |
| [sources/asi-evolve](sources/asi-evolve.md) | Architecture + data + RL algorithm search — vastly out of scope; this is for AI-research-as-research, not scanner optimization |
| [sources/coral](sources/coral.md) | Multi-agent shared-memory co-evolution — heavier than the problem needs; Evo's tree gives most of the benefit at lower complexity |
| [sources/group-evolve](sources/group-evolve.md) | Group-level evolution with shared experience pool — interesting but heavier than needed; Evo + SkillOpt cover the same ground for skills specifically |
| [sources/evoforge](sources/evoforge.md) | Population of full agent.py rewrites — broader scope than skill optimization; Evo's tree-shaped parallelism is the right level |
| [sources/autoagent-hkuds](sources/autoagent-hkuds.md), [sources/autoagent-kevinrgu](sources/autoagent-kevinrgu.md) | Full harness rewrite via dialogue or hill-climb — too broad; you want to optimize *skills*, not regenerate the agent |
| [sources/meta-harness](sources/meta-harness.md) | Closest research analog to skill optimization, but it's a Stanford artifact tied to its specific benchmark; SkillOpt is the productized successor |
| [sources/halo](sources/halo.md) | Production OpenTelemetry traces as the feedback loop — only applies if you have live scanner traffic in production. For an offline experiment with labeled repos, the simpler held-out gate is enough |
| [sources/deep-research](sources/deep-research.md) | GEPA applied to a four-agent research pipeline — same primitive as HonedHaiku, narrower context; HonedHaiku is the better analog for your case |
| [sources/evox](sources/evox.md) | Meta-evolution of the search strategy itself — overkill; Evo's configurable frontier strategy gives you 80% of this at zero cost |
| [sources/autogenesis](sources/autogenesis.md) | Self-modification protocol with typed resources — a governance layer, not an optimizer; relevant if you formalize the experiment infrastructure later |

## The single biggest lever — pick before you pick the optimizer

**The benchmark and metric design matter more than the optimizer choice.**

Vulnerability detection has a brutal precision/recall tension and a long tail of CWEs. The single biggest lever in your setup is whether your failure descriptions are **evidence-bounded** (per [sources/rlm-gepa](sources/rlm-gepa.md)'s contract): *"missed CVE-2023-X in file Y at line N"* or *"false positive: matched `eval(` inside a test fixture at file Z"* gives the optimizer concrete material to act on. A single F1 number does not.

Concretely, set up the metric so that for each evaluation it returns:

1. **Scalar vector**: per-CWE precision/recall + macro-F1 across CWE categories, **plus `$/scan`** (input+output tokens × per-model price, optionally + wall-clock) — return the objectives as a vector, not a pre-collapsed scalar, so the Pareto frontier can see the tradeoff
2. **Per-failure descriptions**: missed findings (which CVE, which file, which line), false positives (which file, which line, what was matched, why it was wrong), **and cost attribution** (what drove the token spend — repeated reads, oversized context, redundant tool calls)
3. **No prescriptive rewrites in the feedback** — the metric names failures, the proposer proposes fixes ([concepts/feedback-signals](concepts/feedback-signals.md) evidence-bounded contract)

This shape is what every system in the *primary fit* list above expects, and it composes with the AgentSpec scope declaration above.

## Suggested experimental design

A concrete plan that uses the primary recommendation:

1. **Split repos**: train / dev (gate) / holdout (final eval) / cross-CWE-holdout (CWE categories never seen during optimization). The cross-CWE-holdout is the generalization probe.
2. **Write the AgentSpec** as in the example above. Commit it to the repo so every optimizer round reads it.
3. **Implement the metric** to return *both* a per-instance scalarization (cost-aware F-β, fed to GEPA's `scores` to drive search) *and* the raw `(precision, recall, $/scan)` per instance (fed to `objective_scores` so GEPA records the objective frontier), plus evidence-bounded per-failure descriptions. No frontier-code patch is needed — GEPA tracks the objective frontier from `objective_scores` (see [multi-objective extension](#multi-objective-extension--optimizing-across-precision-recall-cost-model)); at the end, read it out (or filter `prog_candidate_objective_scores`) and union across runs.
4. **Run `/evo:discover`** on the training set with a seed directive. Let it auto-attach the held-out score floor; **add a cost gate** (reject candidates above a `$/scan` ceiling) alongside it.
5. **Configure the frontier strategy** to `pareto_per_task` (CWE category as the task axis) for specialists, and maintain a separate `(P, R, cost)` objective frontier for operating-point selection — these are two different Paretos.
   - **Sweep `model`** as an outer categorical: one optimization run per model, then pool every `(skill, model)` candidate into the combined objective frontier.
6. **Constrain edit budget** per round (SkillOpt-style) — start at ~3 ops per round, treat this as a hyperparameter, vary it.
7. **Compare against**:
   - An unoptimized hand-written skill (baseline)
   - The [auto-harness](sources/auto-harness.md) single-thread loop on the same skill (single-thread control)
   - The same Evo run with the `argmax` frontier strategy (ablation: does `pareto_per_task` matter?)
   - The same Evo run with no rejected-edit / discarded-hypothesis store (ablation: does negative-signal storage matter?)
8. **Generalization probes** at the end:
   - Score the optimized skill under a different Claude model (transfer probe — SkillOpt-style)
   - Score on the cross-CWE-holdout split
   - Score on a freshly clipped repo from outside the experimental corpus

## Connections

- [concepts/harness-optimization](concepts/harness-optimization.md) — the umbrella concept for everything in scope here
- [concepts/feedback-signals](concepts/feedback-signals.md) — evidence-bounded feedback is the single most important design choice
- [concepts/regression-gating](concepts/regression-gating.md) — held-out-slice gating (Evo auto-attaches it; SkillOpt uses it) is the central anti-overfitting mechanism
- [concepts/evolutionary-optimization](concepts/evolutionary-optimization.md) — Pareto-per-task selection is the right shape for the multi-CWE objective
- [concepts/self-improvement-loop](concepts/self-improvement-loop.md) — the measure → fail → propose → gate cycle is the abstract skeleton you are instantiating
- [overview](overview.md) — wiki-wide synthesis; see especially the *Productive Band* and *Modular Decomposition* sections, both of which apply

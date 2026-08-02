---
title: Experiment — Optimizing Vulnerability-Detection Scanning Workflows
type: analysis
tags: [experiment, vulnerability-detection, security, claude-code-skills, harness-optimization, application]
sources: [skillopt, evo-hq, honedhaiku, optimize-anything, optimize-anything-omni, rlm_gepa, auto-harness, squeeze-evolve]
last_updated: 2026-08-01
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

**model — superseded.** This subsection originally concluded GEPA "cannot do this axis" and recommended an outer per-model sweep. That verdict was wrong: GEPA's candidate is `dict[str, str]` (multi-component), so model assignments can be optimized *inside* a single GEPA run as components, and the "blind to cost/capability structure" objection is defeated by feeding that structure through ASI. The committed design is the **single-loop, no-outer-sweep** approach — see **[Final solution — single-loop GEPA over [prompt, model]](#final-solution--single-loop-gepa-over-prompt-model)** below, which replaces both the outer-sweep recommendation here and the per-model `c_ref` cost normalization in the scalarization subsection (with model as an inner component there is no fixed per-run model, so cost is priced in global dollars).

### Designing the cost-aware scalarization

The scalarization fed to GEPA's `scores` has exactly one job: **drive search toward high precision/recall while treating cost as a real objective — without ever letting "cheap-and-useless" outrank "good-and-expensive."** Everything below follows from that constraint.

#### Fix the instance granularity first

GEPA sums `scores` per evaluation instance, so the unit must be decided before the scalar can be defined:

| Granularity | Quality well-defined? | Cost natural? | Use |
|---|---|---|---|
| `(repo, CWE)` pair | Yes | **No** — one scan covers all CWEs; cost can't be cleanly attributed | per-task (`pareto_per_task`) frontier only |
| **one repo scan** | Yes (macro-F_β over CWEs *within* the repo) | **Yes** — 1 scan = 1 cost | **recommended driver granularity** |

Use **instance = one repo scan**: cost is per-instance with no shared-cost attribution hack, and macro-over-CWE is computed *inside* the instance. The per-CWE breakdown still goes to `objective_scores`, so the per-task frontier keeps working (it reads raw objectives, not the scalar).

#### Quality term — F_β from counts, not from P and R

Per-instance precision/recall are undefined when a repo has no findings or no ground truth. Compute F_β directly from counts to sidestep that:

```
F_β = (1 + β²)·TP / ( (1 + β²)·TP + β²·FN + FP )
```

- Well-defined whenever `TP+FP+FN > 0`; equals `0` when `TP=0` with any error (correctly punishes both do-nothing and spray-everything).
- Convention: a CWE absent from both gold and prediction → exclude it (don't let it inflate the macro); present in gold, nothing reported → contributes `F_β = 0`.
- `Q_i = mean over CWE categories present in repo i of F_β` (macro, matching the metric-design choice above).

**β is the precision/recall dial and a genuine domain decision:** β > 1 favors recall (a missed CVE hurts more than a false positive), β < 1 favors precision (alert fatigue in triage). Security scanning usually leans β ≈ 2 — but β is a *sweep axis*, not a fixed choice (see below).

#### Cost term — normalize per-model, then price as willingness-to-pay

Within a GEPA run the model is fixed (model is the outer sweep), so normalize cost to a **per-model baseline**:

```
ĉ_i = cost_i / c_ref(model)      # c_ref = $/scan of the unoptimized baseline skill on that model
```

Now `ĉ ≈ 1` means "baseline cost," and the **same scalarization weight is comparable across every model in the sweep** — which is what makes the cross-run frontier union legitimate. `cost_i` = Σ over LLM calls of `(in_tok·price_in + out_tok·price_out)` in dollars; for self-hosted open-weight models, `$ = GPU-hours × rate` (keep one currency so frontiers are comparable).

Price it with an exchange rate `V` ("willingness-to-pay"): how many normalized-cost-units you will spend for +1.0 of F. More interpretable than a raw λ.

#### The scalarization

```
score_i = Q_i − ĉ_i / V          subject to hard gate cost_i ≤ B_max
```

Additive (constant exchange rate: a marginal dollar is always worth the same F), with the existing hard cost gate killing anything above `B_max`. **Don't clamp to [0,1]** — a negative score (worse than doing nothing) is meaningful for GEPA's minibatch sums, and the gate already bounds the penalty.

**Calibration recipe** (makes `V` non-arbitrary):
1. `B_max` = max acceptable `$/scan` → normalized `B̂_max = B_max / c_ref`.
2. Decide how much the *worst allowed* candidate may be penalized (e.g. a candidate at the cost ceiling loses at most 0.3 F). Then `V = B̂_max / 0.3`.
   - *Example:* allow up to 3× baseline cost, worst-case penalty 0.3 → `V = 3/0.3 = 10`, so `penalty = ĉ/10`. Baseline-cost candidate loses 0.1 F; a 3× one loses 0.3 F.

This guarantees the max cost penalty stays below 0.5, so the "cheap junk wins" optimum is **structurally impossible**:

| Candidate | Q | ĉ | score (V=10) | Correct? |
|---|---|---|---|---|
| Do-nothing | 0 | ~0.1 | −0.01 | ✓ near-zero |
| Spray everything | ~0.3 | 2.5 | 0.05 | ✓ curbed by cost |
| Good, baseline cost | 0.80 | 1.0 | 0.70 | ✓ wins |
| Good, expensive | 0.82 | 3.0 | 0.52 | ✓ beaten by cheaper-equal |

Quality dominates; cost only decides between comparably-good candidates.

#### The scalarization is a steerable probe, not the answer

This is the concrete form of the exploration-bias mitigation noted above. A single `(β, V)` makes GEPA explore one operating region; **sweep `(β, V)` across runs** (e.g. β ∈ {1, 2}, V ∈ {5, 10, 20}), each pulling GEPA toward a different precision/recall/cost trade-off, then **union all runs' recorded `objective_scores` frontiers** into the final `(P, R, cost)` Pareto. The scalarization steers; the sweep fills out the frontier.

#### What goes where

- → GEPA **`scores`**: the scalar `score_i` (drives search).
- → GEPA **`objective_scores`**: raw `{precision_i, recall_i, cost_usd_i}` per instance, **unscalarized** (this is what the objective frontier is read from and unioned).
- → **`make_reflective_dataset`** (ASI): cost attribution too — *"$0.40/scan, 60% of tokens re-reading files already in context"* — so the proposer can act on cost, not just observe it.

Reference sketch for the adapter:

```python
def per_instance_score(repo_eval, model, beta=2.0, V=10.0, c_ref=None):
    fbetas = [f_beta(c.tp, c.fp, c.fn, beta)          # F_β from counts
              for c in repo_eval.cwes_present]
    Q = sum(fbetas) / len(fbetas)                      # macro over CWEs
    c_hat = repo_eval.cost_usd / c_ref[model]          # per-model normalized
    return Q - c_hat / V                               # gate cost > B_max upstream
```

`β` (fear misses vs. false positives) and `V` (dollars per point of F) are the two domain knobs — defaults β=2, V≈10 via the calibration recipe — and both are the sweep axes, so neither needs to be committed to a single value up front.

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

Keep Evo as the substrate. Run **one GEPA loop with no outer model sweep** (see [Final solution](#final-solution--single-loop-gepa-over-prompt-model) for the full design): each pipeline stage is a **compound `[prompt + model]` component**, so GEPA jointly optimizes prose and per-stage model assignment in a single search, driven by a **cost-aware scalarization** in `scores` (global dollars, since model is now inner) while logging raw `{precision, recall, cost}` to `objective_scores` so GEPA records the objective frontier (confirmed available — see above). Add an explicit **cost gate**; read the objective frontier out at the end — each frontier point carries its own per-stage model mix. The only optional outer loop is over scalarization weights `(β, V)`, unioned into one `(P, R, cost)` frontier. Use **SkillOpt's cross-model transfer** as the property that keeps a stage's prose viable when the optimizer reassigns its model, and SkillOpt-style rejected-edit buffers (over the compound components) as the negative-signal store.

The earlier open question is now **resolved**: GEPA's shipped `scores` is `list[float]` (per-instance scalar), so it does not *search* an objective Pareto — but it *tracks* one via `objective_scores`, so no frontier-code patch is needed. The remaining work is designing the scalarization and unioning frontiers across runs, not extending GEPA's core.

## Secondary fits — useful as control arms or supplements

### [sources/honedhaiku](sources/honedhaiku.md) (Tim Waldin) — closest empirical analog

HonedHaiku applies GEPA to Claude Haiku system prompts for **bug fixing** — the security-shaped sibling of vulnerability detection. It reports +19.7pp on unseen bugs (Haiku 3.5: 65% → 85%) and converged in 4 of 20 allocated iterations.

Two lessons port directly:

- **The Goldilocks band**: prompt-only optimization moved the needle in the ~50–70% baseline range. Below 50%, the model couldn't execute complex methodologies; above ~85%, the prompt was no longer the bottleneck. **Check your baseline detection rate first**: if it's outside this band, expect smaller returns from prompt-only optimization — and lean on SkillOpt's bounded edits, which partially widen the band.
- **Training diversity matters more than iteration count**: a 3-challenge run overfit; 20 challenges across 5 repos generalized. The analog: don't optimize against 3 hand-picked CVEs; sample broadly across CWE categories and repo styles.

### [sources/optimize-anything](sources/optimize-anything.md) (GEPA) — the underlying primitive

GEPA is the search primitive both HonedHaiku and the suggested `pareto_per_task` Evo strategy rest on. If you want to *implement* the optimizer yourself rather than use Evo's packaged form, GEPA is the reference. The core idea — Pareto frontier over multi-metric scoring + LLM-proposed edits + **Actionable Side Information (ASI)** as the feedback channel — is exactly the shape of your problem.

### [sources/optimize-anything-omni](sources/optimize-anything-omni.md) (portfolio of optimizers) — a plateau-breaking ablation, not a replacement

The committed design runs a **single** GEPA loop. `omni` is the external data point arguing that may leave performance on the table: on Frontier-CS — competitive programming, $20/problem, close to this experiment's shape — **no single optimizer dominated**, and racing GEPA + AutoResearch + Meta-Harness then continuing the winner with a *fresh* optimizer beat every standalone (GEPA 43.8→61.8). Notably, all three omni engines are already in scope here: GEPA is the committed inner loop, [Evo](sources/evo.md) is the AutoResearch-style substrate, and [Meta-Harness](sources/meta-harness.md) is the research analog SkillOpt productized.

Two reasons to treat it as an **ablation arm**, not a wholesale swap:

- **The frontier readout needs adapting.** omni's reported protocol keeps a single "best" candidate at the phase boundary; this experiment needs the full `(P, R, cost)` objective frontier. The frontier-preserving version of omni's idea is exactly what the `(β, V)` scalarization sweep already does — race per scalarization, union the frontiers.
- **The cheap, model-agnostic half is free to try.** omni's most portable finding is "a fresh optimizer breaks plateaus." Concretely: keep the committed GEPA loop but, when it plateaus, **reseed a fresh GEPA instance from the current best** before spending the rest of the budget. That costs nothing structurally and is the first thing to test before adding whole other engines.

This does **not** overturn the [Final solution](#final-solution--single-loop-gepa-over-prompt-model) — it adds a reseed-on-plateau step and a portfolio ablation arm (below).

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

### [sources/squeeze-evolve](sources/squeeze-evolve.md) — cost-aware model routing as a scan-time lever

Squeeze-Evolve is the other inference-time option, and it speaks directly to the **cost** objective. Its idea — route each *problem* to a cheap or expensive model by a zero-cost difficulty proxy (self-confidence / answer diversity) — maps onto scanning as: run cheap models on obviously-clean files and escalate to an expensive model only where the cheap model is *uncertain*. This is the per-instance, verifier-free cousin of the [Final solution](#final-solution--single-loop-gepa-over-prompt-model)'s per-*stage* `[prompt, model]` search: the Final Solution decides model assignment **at optimization time** (baked into the artifact); Squeeze-Evolve decides it **at scan time** (per repo/file, from live confidence). They compose — the optimizer can fix per-stage models while a confidence router escalates hard inputs within a stage — but treat scan-time routing as a *cost lever inside the cost objective* (like AutoReason), not as an optimizer choice, and beware that a confidently-wrong scan is mis-routed as "easy."

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
   - **Make each stage a compound `[prompt + model]` component** so GEPA optimizes the per-stage model assignment *inside* the single loop — no outer model sweep (see [Final solution](#final-solution--single-loop-gepa-over-prompt-model)). Enforce model-set membership in `evaluate` (clamp + penalize out-of-set ids), and feed per-stage model context (current model, realized cost, in-set cheaper/dearer alternatives) through ASI so the proposer can flip model and adjust prose in one edit. Each objective-frontier point then carries its own per-stage model mix.
6. **Constrain edit budget** per round (SkillOpt-style) — start at ~3 ops per round, treat this as a hyperparameter, vary it.
7. **Compare against**:
   - An unoptimized hand-written skill (baseline)
   - The [auto-harness](sources/auto-harness.md) single-thread loop on the same skill (single-thread control)
   - The same Evo run with the `argmax` frontier strategy (ablation: does `pareto_per_task` matter?)
   - The same Evo run with no rejected-edit / discarded-hypothesis store (ablation: does negative-signal storage matter?)
   - **Reseed-on-plateau** and an **`omni`-style portfolio arm** ([optimize-anything-omni](sources/optimize-anything-omni.md)): (a) reseed a fresh GEPA instance from the current best when a run plateaus; (b) race the GEPA loop against a Claude-Code AutoResearch run on the same tasks and continue the winner — does portfolio+continue beat the single GEPA loop at equal budget?
8. **Generalization probes** at the end:
   - Score the optimized skill under a different Claude model (transfer probe — SkillOpt-style)
   - Score on the cross-CWE-holdout split
   - Score on a freshly clipped repo from outside the experimental corpus

## Final solution — single-loop GEPA over [prompt, model]

This is the consolidated, committed design. **No outer model sweep.** One GEPA run jointly optimizes the per-stage prompts *and* their model assignments against a `(precision, recall, cost)` objective, driven by a cost-aware scalar and steered by ASI that carries both detection failures and cost/performance attribution. (This supersedes the earlier outer-sweep framing under [Does GEPA handle this out of the box?](#does-gepa-handle-this-out-of-the-box) and the per-model cost normalization in [Designing the cost-aware scalarization](#designing-the-cost-aware-scalarization).)

### 1. Candidate layout — one compound component per stage

GEPA's candidate is `dict[str, str]` ("a mapping from component names to component text" — verified against `gepa-ai/gepa`: `api.py`, `core/adapter.py`). Make each pipeline stage a single **compound** component whose text holds both the prompt and a model directive:

```
seed_candidate = {
  "triage":   "<prompt prose ...>\n\n[model: claude-opus-4-8]",
  "locate":   "<prompt prose ...>\n\n[model: claude-haiku-4-5]",
  "classify": "<prompt prose ...>\n\n[model: claude-haiku-4-5]",
  "confirm":  "<prompt prose ...>\n\n[model: claude-opus-4-8]",
}
```

Why prompt+model in *one* component, not two: GEPA's `RoundRobinReflectionComponentSelector` edits **one component per round** (verified — the selectors are deterministic round-robin or all-at-once, not LLM-chosen). Separate `stage.prompt` / `stage.model` slots would let the optimizer flip a stage's model on one round and not re-adapt its prompt until a later round — and the candidate gets rejected in between, under the now-mismatched prompt (the prompt×model entanglement problem). Folding both into one component means a single mutation **co-edits prose and model together**, while keeping round-robin's focused, per-stage reflection. The adapter parses the `[model: ...]` directive out at evaluation time to wire the pipeline.

This is the per-stage heterogeneity the whole design exists for: the optimizer can land on **opus-for-triage / haiku-for-extraction**, which an outer single-model sweep structurally cannot express.

### 2. Allowed model set is bounded by the execution layer

A model directive may only name a model the runner can actually execute: closed-weight Claude models run natively; open-weight models run on Evo's non-Claude backends (modal/e2b/daytona/aws). So `ALLOWED = {claude-*} ∪ {open-weight models the backends serve}`. Enforce membership in `evaluate`: parse the directive, and if it names a model outside the set, clamp to the stage's baseline model **and** apply a hard penalty — hallucinated ids never crash a run and never look attractive. Put `ALLOWED` + the per-model price table into the reflective dataset so the proposer stays in-set by construction.

### 3. ASI carries cost/performance, not just detection failures

For each component being updated, `make_reflective_dataset(candidate, eval_batch, components_to_update)` returns three kinds of evidence-bounded signal:

- **Detection failures attributable to this stage** — "missed CVE-2023-X at file Y:N", "false positive: matched `eval(` in a test fixture at Z" (the existing evidence-bounded contract — [concepts/feedback-signals](concepts/feedback-signals.md)).
- **Cost attribution** — "$0.41/scan; 60% of tokens re-reading files already in context; 3 redundant tool calls."
- **Model context for this stage** — current model and its realized cost/latency this run, the miss/FP patterns tied to it, and the cheaper/dearer in-set alternatives with their prices.

The third item is what makes joint `[prompt, model]` search work: the proposer can reason *"this stage is simple extraction and the haiku run had no misses here — downgrade the model and tighten the prompt"* in **one** edit, instead of treating model as an opaque bandit arm. This is the concrete answer to the "GEPA is blind to the model's cost/capability structure" objection: you hand it that structure.

### 4. Metric — scalar to drive search, raw vector to record the frontier

Per instance (= one repo scan), the adapter returns:

- → GEPA **`scores`** (`list[float]`): the cost-aware scalar `score_i` that drives search.
- → GEPA **`objective_scores`**: the raw `{precision_i, recall_i, cost_usd_i}` vector, **unscalarized** — GEPA records this as the objective frontier without consuming it for selection (verified: `objective_scores` is stored/aggregated only, never read by frontier-membership logic). Each recorded frontier point carries its candidate's full **per-stage model assignment**.

### 5. Scalarization and calibration — global dollars (no per-model normalization)

Because model is now an **inner** component (mutated within the single run), there is no fixed per-run model, so the earlier per-model `c_ref` normalization no longer applies. Price cost in one global currency — dollars — directly:

```
Q_i     = mean over CWEs present in repo i of  F_β,   F_β = (1+β²)·TP / ((1+β²)·TP + β²·FN + FP)
score_i = Q_i − cost_i / V              subject to hard gate  cost_i ≤ B_max
```

- `cost_i` = Σ over all LLM calls in the scan of `(in_tok·price_in + out_tok·price_out)`, summed across **whatever per-stage models the candidate chose**; self-hosted open-weight cost = GPU-hours × rate, same currency so frontiers stay comparable.
- `V` = willingness-to-pay in **$ per F-point**.
- `β` = recall/precision dial (security ≈ 2; a missed CVE hurts more than a false positive).

**Calibration** (keeps "cheap-and-useless wins" structurally impossible):
1. `B_max` = max acceptable `$/scan` (the hard gate).
2. Pick the worst penalty a ceiling-cost candidate may take (e.g. 0.3 F). Then `V = B_max / 0.3`.
   - *Example:* baseline opus scan ≈ $0.20, allow up to `B_max = $0.60`, worst-case penalty 0.3 → `V = 0.60 / 0.3 = $2.0` per F-point → `penalty = cost / 2.0`. A $0.20 scan loses 0.10 F; a $0.60 scan loses 0.30 F.

Degenerate-optima check (note the heterogeneous-model candidate **winning** — the design goal):

| Candidate | Q | cost ($) | score (V=2.0) | Correct? |
|---|---|---|---|---|
| Do-nothing (cheapest models, no work) | 0 | 0.02 | −0.01 | ✓ near-zero |
| Spray everything | 0.30 | 0.50 | 0.05 | ✓ curbed by cost |
| **Good, cheap-model mix (haiku on easy stages)** | **0.78** | **0.15** | **0.705** | **✓ wins** |
| Good, all-opus | 0.82 | 0.55 | 0.545 | ✓ beaten by cheaper-equal |

Quality dominates; cost only decides between comparably-good candidates — and the per-stage heterogeneous candidate is the winner, which is the entire point of the inner-model design.

### 6. Loop, gates, and readout

- **Selector:** `RoundRobinReflectionComponentSelector` over the compound stage components.
- **Gates:** hard cost gate `cost_i ≤ B_max`; held-out-repo score-floor gate (Evo auto-attaches at discovery — [concepts/regression-gating](concepts/regression-gating.md)). Gates hard-veto over score.
- **Two Paretos, unchanged:** `pareto_per_task` for CWE specialists (reads raw objectives), plus the `(P, R, $/scan)` objective frontier read from `objective_scores` at the end. Each objective-frontier point now also names its **per-stage model mix** — the deliverable is "pick an operating point *and* its model assignment per deployment budget."
- **The only remaining outer loop is over scalarization weights, not model:** a single `(β, V)` steers GEPA to one region; an optional small `(β, V)` sweep across a few runs + union of recorded objective frontiers fills the frontier out (the exploration-bias mitigation).

### 7. Reference adapter sketch

```python
ALLOWED = {"claude-opus-4-8", "claude-haiku-4-5", "qwen2.5-coder-32b", ...}
PRICE   = {m: (price_in, price_out) for m in ALLOWED}   # $/token; open-weight via GPU-hours
BETA, V, B_MAX = 2.0, 2.0, 0.60

def parse_stage(text):
    prose, model = split_model_directive(text)           # pulls "[model: ...]" out of the component
    return prose, model

def evaluate(self, batch, candidate, capture_traces=False):
    stages = {name: parse_stage(t) for name, t in candidate.items()}
    penalty = sum(MODEL_VIOLATION for _, m in stages.values() if m not in ALLOWED)  # validity
    scores, objs, traces = [], [], []
    for repo in batch:
        ev = run_pipeline(repo, stages)                  # realized counts, $cost, per-stage attribution
        if ev.cost_usd > B_MAX:                          # hard cost gate
            scores.append(GATE_FAIL); objs.append(ev.objective_vector()); continue
        Q = mean(f_beta(c.tp, c.fp, c.fn, BETA) for c in ev.cwes_present)
        scores.append(Q - ev.cost_usd / V - penalty)
        objs.append({"precision": ev.P, "recall": ev.R, "cost_usd": ev.cost_usd})
        if capture_traces: traces.append(ev.trace)
    return EvaluationBatch(scores=scores, objective_scores=objs, trajectories=traces or None)

def make_reflective_dataset(self, candidate, eval_batch, components_to_update):
    # per stage in components_to_update, emit: stage-attributable misses/FPs, cost attribution,
    # current model + price, and in-set cheaper/dearer alternatives so the proposer can flip model + prose together
    ...
```

`β` and `V` are the two domain knobs (defaults β=2, V≈2.0 via the calibration recipe); **the per-stage model assignment is part of the optimized artifact, not a hyperparameter you set.**

## Validation against source (2026-07-03)

Re-verified the load-bearing claims above against the current `gepa-ai/gepa` checkout (commit `92dadff`, v0.1.1) and the existing application (`ai-sast-benchmark-A1b3/autoresearch/gepa/`). Consolidated implementation plan: `ai-sast-benchmark-A1b3/docs/superpowers/plans/2026-07-03-multi-objective-gepa.md`.

**Confirmed:** candidate is `dict[str, str]` (`core/adapter.py:12`); `EvaluationBatch.scores` is per-instance `list[float]` summed for minibatch acceptance; `RoundRobinReflectionComponentSelector` mutates exactly one component per iteration (so the compound `[prompt + model]` component argument holds); instance-Pareto frontier per validation id; the `optimize_anything` evaluator contract `(score, side_info)` matches the existing SAST optimizer unchanged.

**Superseded — GEPA moved past the 2026-06-23 verification.** The claim that `objective_scores` is "stored/aggregated only, never read by frontier-membership logic" is now **wrong**:

- `GEPAState` has a `frontier_type ∈ {instance, objective, hybrid, cartesian}` parameter, and `ParetoCandidateSelector` samples parents from `state.get_pareto_front_mapping()`, which **does** consume the objective frontier under `objective`/`hybrid`/`cartesian`. In `optimize_anything`, `EngineConfig.frontier_type` **defaults to `"hybrid"`** (instance + objective champions).
- Consequence: the exploration-bias mitigation is weaker than claimed to be necessary — per-objective champions (best-precision, best-recall, best-neg-cost holders) stay alive as parents natively. The `(β, V)` sweep + frontier union remains useful for *filling out* the frontier but is no longer the only mechanism keeping cost-favored candidates explorable.
- Caveat that survives: the stored objective frontier keeps **per-axis champions only**, not the full non-dominated set over the joint `(P, R, cost)` vector — the final readout still needs your own dominance filter over `prog_candidate_objective_scores` (the plan adds `frontier.py` for this).

**New facts the earlier analysis missed:**

- **Higher-is-better applies to objectives too.** Frontier updates keep maxima, so cost must be fed as `neg_cost_usd`, never raw `cost_usd` — feeding raw cost would make the frontier reward *expensive* candidates.
- **`side_info["scores"]` is the delivery mechanism.** In `optimize_anything`, a `"scores"` dict in the evaluator's side_info is extracted into `objective_scores` automatically (and `"<component>_specific_info"` scores become namespaced `component::metric` objectives) — no custom `GEPAAdapter` needed.
- **`AcceptanceCriterion` is pluggable and receives per-instance `objective_scores`** — the hard cost gate can veto over score without patching core (plan: `CostGateAcceptance`).
- **`on_candidate_rejected` / `on_proposal_start` / `on_proposal_end` callbacks exist**, giving a clean seam for the SkillOpt-style rejected-edit buffer (plan: `RejectedEditBuffer`, fed back through side_info).
- **The scan runner already does the cost plumbing**: `claude-codex-sast/run.py` accepts `--model` and writes `cost_usd` / `model_usage` into `meta.json` from a maintained `MODEL_PRICING` table — "you write the cost metric" reduces to reading a file.

**Bounded edits and negative-feedback memory (checked 2026-07-03) — both achievable without core changes, via different seams:**

- **Full rewrites are structural in the default path.** `InstructionProposalSignature.output_extractor` (`strategies/instruction_proposal.py`) takes the LM's fenced block as the *complete replacement text* for the component — the default proposer is one-shot full regeneration by construction. `reflection_prompt_template` can *ask* for restraint (soft discipline, already in the plan) but cannot change the extraction contract; a "return only 3 edit ops" instruction would just produce a broken component.
- **The hard seam is `custom_candidate_proposer`** (`ReflectionConfig`, checked in `reflective_mutation.propose_new_texts` before the default path): signature `(candidate, reflective_dataset, components_to_update) -> dict[str, str]`, full control. A SkillOpt-style bounded proposer lives entirely here: prompt your own LM for structured anchored ops (`ADD/REPLACE/DELETE`, max N per round — the "textual learning rate" as an app-layer hyperparameter), apply them programmatically, re-ask or return the parent text on budget/parse violation (an identical candidate is safely rejected by strict-improvement acceptance). Caveats: it bypasses GEPA's objective/background template (the proposer must embed that context itself), and its LM calls are invisible to the `max_reflection_cost` stopper unless it reuses the same tracked LM.
- **Core keeps no memory of rejected candidates** — `GEPAState` stores accepted programs only; rejections surface solely through callbacks (`on_proposal_start` carries the parent text, `on_proposal_end` the proposed text, `on_candidate_rejected` the verdict — pairing all three reconstructs the failed edit). Two app-layer closures: (a) callback buffer → injected via `side_info["recent_rejected_edits"]` (in the plan, Task 7); (b) with a custom proposer, inject the buffer directly into the proposal prompt — cleaner, since the negative signal reaches the proposer without transiting the evaluator.
- The two features **compose naturally in one custom proposer** (bounded ops + rejected-ops memory = the full SkillOpt mechanism, including "don't re-try ops similar to rejected ones"). Deferred design note appended to the implementation plan.

**Reality check on the Final Solution's pipeline framing:** the deployed scanner is a **single-stage** Claude Code session driven by one `SKILL.md` — the four-stage `triage/locate/classify/confirm` candidate layout has no staged harness to execute it yet. The committed near-term shape is therefore **one** compound component `{"sast_skill": "<SKILL.md prose>\n\n[model: <id>]"}` (model applies to the whole scan via `--model`). Allowed model set today is Claude-CLI-runnable models (`opus-4-6`, `sonnet-4-6`, `haiku-4-5`); open-weight models need a non-Claude execution path before they can enter the set.

**Per-stage models — path resolved (2026-07-03, follow-up check):** the anticipated runner surgery mostly **dissolves**. Claude Code natively carries per-stage model heterogeneity via project subagents: `.claude/agents/<stage>.md` files support a `model:` frontmatter field (aliases `haiku`/`sonnet`/`opus`, full ids, or `inherit`), headless `claude -p` loads them from cwd like interactive mode, and the runner already (a) executes with `cwd = harness workspace`, (b) allows the `Agent` tool, and (c) persists the result event's per-model-id `modelUsage` token breakdown into `meta.json` — so per-stage cost attribution is free whenever stages use distinct models. **Zero mandatory runner changes.** The real Phase-2 work is: authoring a staged seed harness (orchestrator SKILL.md + stage agent files) that must first match the monolithic baseline within noise; extending the candidate to multi-component (`stage_<name>` components, each compound prose + `[model:]`, rendered into adapter-owned frontmatter so `name`/`description`/`tools` stay out of the optimizer's reach per the AgentSpec); and per-stage ASI via `optimize_anything`'s native `<component>_specific_info` merge. Round-robin then delivers exactly the per-stage co-edit of prose+model that this Final Solution specifies. One empirical check remains (undocumented): that subagent tokens appear in `modelUsage` — one instrumented scan settles it. Full task breakdown: plan Phase 2 in `ai-sast-benchmark-A1b3/docs/superpowers/plans/2026-07-03-multi-objective-gepa.md`.

## Connections

- [concepts/harness-optimization](concepts/harness-optimization.md) — the umbrella concept for everything in scope here
- [concepts/feedback-signals](concepts/feedback-signals.md) — evidence-bounded feedback is the single most important design choice
- [concepts/regression-gating](concepts/regression-gating.md) — held-out-slice gating (Evo auto-attaches it; SkillOpt uses it) is the central anti-overfitting mechanism
- [concepts/evolutionary-optimization](concepts/evolutionary-optimization.md) — Pareto-per-task selection is the right shape for the multi-CWE objective
- [concepts/self-improvement-loop](concepts/self-improvement-loop.md) — the measure → fail → propose → gate cycle is the abstract skeleton you are instantiating
- [overview](overview.md) — wiki-wide synthesis; see especially the *Productive Band* and *Modular Decomposition* sections, both of which apply

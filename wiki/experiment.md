---
title: Experiment — Optimizing Vulnerability-Detection Scanning Workflows
type: analysis
tags: [experiment, vulnerability-detection, security, claude-code-skills, harness-optimization, application]
sources: [skillopt, evo-hq, honedhaiku, optimize-anything, rlm_gepa, auto-harness]
last_updated: 2026-06-06
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
| Model | Closed-weight (Claude Code) — no weight updates available |

Because the model is closed-weight and the artifact is a text-level skill, the entire wiki section on **weight optimization** ([[sources/agentflow]], [[sources/trace]], [[sources/skill-rl-skill0]]) is out of scope from the start. The relevant literature is **harness optimization** + **text-artifact optimization**.

## Primary recommendation

### [[sources/skillopt]] (Microsoft Research) + [[sources/evo]] (evo-hq), combined

Use **Evo** as the orchestration substrate and **SkillOpt's discipline** as the proposal style inside it.

#### Why SkillOpt

It is the only system in the wiki that *literally* treats a skill document (`best_skill.md`) as the trainable state of a frozen LLM. The mechanism matches your artifact exactly:

- **Bounded add/delete/replace edits** ("textual learning rate") prevents the optimizer from rewriting a working skill to chase a marginal failure — important when vuln-detection knowledge accumulates across many CWE categories
- **Held-out validation gate** on every edit (matches your need for a held-out repo split)
- **Rejected-edit buffer** as a persistent negative-signal store — when an edit fails the gate, it is retained as structured "what not to try" rather than discarded; the optimizer reads from this buffer on future rounds
- **Best-or-tied-best on 52/52 settings** across 7 models × 6 benchmarks, with the strongest cross-model (+15.2%) and cross-harness (+31.8%) transfer evidence in the wiki — directly relevant if you want the optimized skill to keep working when you change the underlying model (Sonnet → Opus, or vice versa)

The "textual learning rate" framing is the load-bearing intuition: **a smaller edit budget widens the productive band** ([[concepts/regression-gating]]). Vuln detection is a domain where you want the optimizer to *refine* a careful checklist of patterns, not regenerate the prompt from scratch every round.

#### Why Evo

Of all the orchestrators in the wiki, Evo is the only one that **ships as Claude Code skills** (`/evo:discover`, `/evo:optimize`) and is structurally tailored to "optimize a skill against a benchmark in a repo." The map onto your problem:

| Your setup | Evo primitive |
|------------|---------------|
| "Define what to scan and how to score it" | `/evo:discover` — interactive setup of benchmark + metric direction; can be seeded with a one-line directive |
| "Don't overfit to my labeled repos" | **Held-out-slice score-floor gate auto-attached at discovery** — generalization protection is the default, not something you have to remember to wire up |
| "Run experiments in parallel" | Subagents in isolated git worktrees |
| "Maintain specialists across CWE categories" | `pareto_per_task` frontier strategy — credited to [[sources/optimize-anything|GEPA]]; keeps branches that win on specific CWEs even when their aggregate lags |
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

## Secondary fits — useful as control arms or supplements

### [[sources/honedhaiku]] (Tim Waldin) — closest empirical analog

HonedHaiku applies GEPA to Claude Haiku system prompts for **bug fixing** — the security-shaped sibling of vulnerability detection. It reports +19.7pp on unseen bugs (Haiku 3.5: 65% → 85%) and converged in 4 of 20 allocated iterations.

Two lessons port directly:

- **The Goldilocks band**: prompt-only optimization moved the needle in the ~50–70% baseline range. Below 50%, the model couldn't execute complex methodologies; above ~85%, the prompt was no longer the bottleneck. **Check your baseline detection rate first**: if it's outside this band, expect smaller returns from prompt-only optimization — and lean on SkillOpt's bounded edits, which partially widen the band.
- **Training diversity matters more than iteration count**: a 3-challenge run overfit; 20 challenges across 5 repos generalized. The analog: don't optimize against 3 hand-picked CVEs; sample broadly across CWE categories and repo styles.

### [[sources/optimize-anything]] (GEPA) — the underlying primitive

GEPA is the search primitive both HonedHaiku and the suggested `pareto_per_task` Evo strategy rest on. If you want to *implement* the optimizer yourself rather than use Evo's packaged form, GEPA is the reference. The core idea — Pareto frontier over multi-metric scoring + LLM-proposed edits + **Actionable Side Information (ASI)** as the feedback channel — is exactly the shape of your problem.

### [[sources/rlm-gepa]]'s `AgentSpec` — adopt the *idea* even if not the framework

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

### [[sources/auto-harness]] (NeoSigma) — single-thread baseline

The closest single-thread control arm: agent edits its own harness, gates with an 80% regression threshold, keeps a `learnings.md` between sessions. If you want a *baseline* that does not depend on Evo's tree-search or SkillOpt's bounded edits, this is the simplest reproducible loop in the wiki.

## What about scan-time refinement?

### [[sources/autoreason]] — orthogonal, not core, but worth considering at scan time

AutoReason's per-query tournament (incumbent vs. adversarial revision vs. synthesis, with a blind Borda-count judge panel) is an **inference-time** loop. It does not optimize the skill. But it could resolve a different problem inside your scanner: *"Is this candidate finding actually a vulnerability or a false positive?"* The tournament gating addresses three pathologies that show up in naive critique-revise (prompt bias, scope creep, lack-of-restraint) — all of which are real in security triage.

Use it as a **separate axis** in your workflow comparison, not as an optimizer choice.

## What to skip and why

| Skipped | Why |
|---------|-----|
| [[sources/agent0]] | Two-agent bootstrap from zero data — you have ground truth, this isn't the right tool |
| [[sources/agentflow]] | RL on planner weights — Claude is closed-weight |
| [[sources/trace]] | LoRA adapters per capability — same closed-weight problem |
| [[sources/skill-rl-skill0]] | RL training; closed-weight |
| [[sources/asi-evolve]] | Architecture + data + RL algorithm search — vastly out of scope; this is for AI-research-as-research, not scanner optimization |
| [[sources/coral]] | Multi-agent shared-memory co-evolution — heavier than the problem needs; Evo's tree gives most of the benefit at lower complexity |
| [[sources/group-evolve]] | Group-level evolution with shared experience pool — interesting but heavier than needed; Evo + SkillOpt cover the same ground for skills specifically |
| [[sources/evoforge]] | Population of full agent.py rewrites — broader scope than skill optimization; Evo's tree-shaped parallelism is the right level |
| [[sources/autoagent-hkuds]], [[sources/autoagent-kevinrgu]] | Full harness rewrite via dialogue or hill-climb — too broad; you want to optimize *skills*, not regenerate the agent |
| [[sources/meta-harness]] | Closest research analog to skill optimization, but it's a Stanford artifact tied to its specific benchmark; SkillOpt is the productized successor |
| [[sources/halo]] | Production OpenTelemetry traces as the feedback loop — only applies if you have live scanner traffic in production. For an offline experiment with labeled repos, the simpler held-out gate is enough |
| [[sources/deep-research]] | GEPA applied to a four-agent research pipeline — same primitive as HonedHaiku, narrower context; HonedHaiku is the better analog for your case |
| [[sources/evox]] | Meta-evolution of the search strategy itself — overkill; Evo's configurable frontier strategy gives you 80% of this at zero cost |
| [[sources/autogenesis]] | Self-modification protocol with typed resources — a governance layer, not an optimizer; relevant if you formalize the experiment infrastructure later |

## The single biggest lever — pick before you pick the optimizer

**The benchmark and metric design matter more than the optimizer choice.**

Vulnerability detection has a brutal precision/recall tension and a long tail of CWEs. The single biggest lever in your setup is whether your failure descriptions are **evidence-bounded** (per [[sources/rlm-gepa]]'s contract): *"missed CVE-2023-X in file Y at line N"* or *"false positive: matched `eval(` inside a test fixture at file Z"* gives the optimizer concrete material to act on. A single F1 number does not.

Concretely, set up the metric so that for each evaluation it returns:

1. **Scalar**: per-CWE precision/recall + macro-F1 across CWE categories
2. **Per-failure descriptions**: missed findings (which CVE, which file, which line) and false positives (which file, which line, what was matched, why it was wrong)
3. **No prescriptive rewrites in the feedback** — the metric names failures, the proposer proposes fixes ([[concepts/feedback-signals]] evidence-bounded contract)

This shape is what every system in the *primary fit* list above expects, and it composes with the AgentSpec scope declaration above.

## Suggested experimental design

A concrete plan that uses the primary recommendation:

1. **Split repos**: train / dev (gate) / holdout (final eval) / cross-CWE-holdout (CWE categories never seen during optimization). The cross-CWE-holdout is the generalization probe.
2. **Write the AgentSpec** as in the example above. Commit it to the repo so every optimizer round reads it.
3. **Implement the metric** to return evidence-bounded per-failure descriptions, not just scalars.
4. **Run `/evo:discover`** on the training set with a seed directive. Let it auto-attach the held-out score floor.
5. **Configure the frontier strategy** to `pareto_per_task` (CWE category as the task axis).
6. **Constrain edit budget** per round (SkillOpt-style) — start at ~3 ops per round, treat this as a hyperparameter, vary it.
7. **Compare against**:
   - An unoptimized hand-written skill (baseline)
   - The [[sources/auto-harness|auto-harness]] single-thread loop on the same skill (single-thread control)
   - The same Evo run with the `argmax` frontier strategy (ablation: does `pareto_per_task` matter?)
   - The same Evo run with no rejected-edit / discarded-hypothesis store (ablation: does negative-signal storage matter?)
8. **Generalization probes** at the end:
   - Score the optimized skill under a different Claude model (transfer probe — SkillOpt-style)
   - Score on the cross-CWE-holdout split
   - Score on a freshly clipped repo from outside the experimental corpus

## Connections

- [[concepts/harness-optimization]] — the umbrella concept for everything in scope here
- [[concepts/feedback-signals]] — evidence-bounded feedback is the single most important design choice
- [[concepts/regression-gating]] — held-out-slice gating (Evo auto-attaches it; SkillOpt uses it) is the central anti-overfitting mechanism
- [[concepts/evolutionary-optimization]] — Pareto-per-task selection is the right shape for the multi-CWE objective
- [[concepts/self-improvement-loop]] — the measure → fail → propose → gate cycle is the abstract skeleton you are instantiating
- [[overview]] — wiki-wide synthesis; see especially the *Productive Band* and *Modular Decomposition* sections, both of which apply

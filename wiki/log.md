# Wiki Log

Append-only record of ingests, queries, and lint passes.
Format: `## [YYYY-MM-DD] type | description`

## [2026-07-31] ingest | OPHIS + optimize_anything Omni + Squeeze-Evolve + Self-Evolving taxonomy → 4 new source pages

Four sources ingested together (each lands in a different part of the wiki):

**optimize_anything Omni** (GEPA team, blog 2026-07-22, `optimize-anything-omni`):
- `optimize_anything` becomes **engine-pluggable** (`engine=gepa|autoresearch|meta_harness`) and **pipeline-composable**. The three engines are three systems already in the wiki: [GEPA](sources/optimize-anything.md), [AutoResearch](sources/autoresearch-vs-hpo.md), [Meta-Harness](sources/meta-harness.md).
- **omni** meta-optimizer = 2-phase: explore (race all engines in parallel on a shared budget, keep best) → continue (seed a *fresh* optimizer with the winner to break plateaus). Composition helpers: `optimize_sequential/best_of/parallel/vote/adaptive_sequential`. **Terrarium** = fair-comparison harness (pins tasks/budget/model = Sonnet 4.6 medium thinking).
- Frontier-CS (10 problems, $20 each): **no single optimizer dominates**, but every omni variant beats every standalone (GEPA 43.8→61.8 +41%; omni-AutoResearch 63.2 best).

**Squeeze-Evolve** (COLM 2026, `squeeze-evolve`):
- Verifier-free evolutionary **test-time scaling**; population = candidate *answers*. 5-stage loop Score→Select→Route→Recombine→Update; routes each refinement step to a cheap/expensive model by **per-instance difficulty** (confidence via logprobs / answer diversity percentiles). Inspired by RSA; cites OpenEvolve. AIME25/HMMT25/GPQA-Diamond; NVIDIA Dynamo + Claude Code plugin. Same-or-better accuracy at a fraction of the cost (no printed numbers).
- Contrast baked in: ShinkaEvolve routes the *mutation operator* by cost-UCB; Squeeze-Evolve routes the *per-instance answer refinement* by difficulty — two layers of the same `[model]` idea.

**OPHIS** (MetaCircle / Ziming Liu, blog Jul 2026, `ophis`):
- The **contrarian**: mechanistic auto-research, **no LLM, no evolution**. Loop = Observation→Problem→Hypothesis→Intervention→Speed-up over ~6,000 tensor-level training-dynamics observables; *deduces* interventions from causal hypotheses instead of searching.
- Results: grokking 72.9% substantial-improve / 13.7% fail vs LLM 57.9% / 42.1% (350 tricks); NanoGPT val BPB 0.93410→0.93184 = **−7.43σ** on an already-RSI-optimized baseline. Discovered a novel "forking" phenomenon.
- Carries its **own** Stage 1/2/3 taxonomy (causal depth: internet-prior/Karpathy → RSI statistical memory → mechanistic) — flagged everywhere as *not* CORAL's Stage 1/2/3 (autonomy) taxonomy.

**Self-Evolving Agents** (Xinming Tu, blog 2026-07-22, `self-evolving`):
- Taxonomy, not a system. **3×3 matrix**: What evolves (External Files / Agent Harness / Model Weights) × When it persists (Single Session / Across Sessions / Across Users). **Consolidation path** files→harness→weights (durability & update-cost rise). Recursive self-improvement = the loop applied to AI development itself.
- Used as a wiki-wide lens: mapped existing wiki systems onto the matrix (densest in Harness × Across-Sessions; sparse in Across-Users).

Wiki updates:
- 4 new source pages under `wiki/sources/`.
- `index.md`: 4 rows in Source Summaries + 4 in Source File Map; last_updated bumped.
- `concepts/evolutionary-optimization.md`: Squeeze-Evolve (cost-routed test-time population) + omni (portfolio of optimizers) sections; 2 new hierarchy-of-evolution rows; frontmatter.
- `concepts/feedback-signals.md`: OPHIS (mechanistic internal-dynamics = new richest tier) + Squeeze-Evolve (verifier-free difficulty proxy) named concepts; frontmatter.
- `concepts/self-improvement-loop.md`: OPHIS mechanistic non-search loop + taxonomy-collision warning; note pointing to Tu's What×When cut; frontmatter.
- `concepts/harness-optimization.md`: omni subsection + comparison row + connections; Tu "Agent Harness" substrate; frontmatter.
- `concepts/regression-gating.md`: OPHIS plausibility+variance (pre-eval) gating; Squeeze-Evolve verifier-free counterexample; frontmatter.
- `concepts/knowledge-accumulation.md`: "Consolidation Path files→harness→weights" section (Tu) reorganizing the existing stores; frontmatter.
- `overview.md`: What-Can-Be-Optimized rows (OPHIS, AlphaEvolve/ShinkaEvolve backfill, Squeeze-Evolve, omni) + Tu cross-cut note; feedback-signal bullets; 2 new loop-architecture subsections (omni portfolio, OPHIS mechanistic); 6 empirical-results rows (backfilled AlphaEvolve/ShinkaEvolve too); 4 open questions; See-Also; frontmatter (also added alphaevolve/shinkaevolve, missing since the prior ingest).
- `experiment.md`: "omni portfolio beats single optimizer — ablation arm, not a replacement" subsection (does NOT overturn the committed single-GEPA-loop decision: different objective shape, orthogonal to the `[prompt,model]` candidate layout, budget already partly covered by Evo's (β,V) sweep); added portfolio ablation to the experimental design + connections; frontmatter.

Key positional claims:
- omni is the top of the "hierarchy of evolution" — optimizing the *portfolio of optimizers*; its "no single optimizer dominates" is EvoX's "no single strategy dominates" lifted one level, and the strongest external tension with experiment.md's single-loop commitment (added as an ablation arm, not a plan change).
- OPHIS is the wiki's clearest mechanistic-understanding alternative to search-based self-improvement, and pushes the feedback-signal spectrum a layer deeper (internal training dynamics). Its observable-design-carries-the-value finding suggests those observables could feed an LLM/evolutionary proposer as ASI.
- Squeeze-Evolve + ShinkaEvolve together are the cleanest illustration of two layers at which `[model]` becomes a search coordinate (per-instance difficulty vs. operator cost-UCB) — relevant to experiment.md.
- Tu's What×When taxonomy is the best external organizing lens for the whole wiki; folded into overview.md and the loop/harness/knowledge concept pages.

## [2026-07-03] query | validation of experiment.md final solution against current GEPA source + consolidated implementation plan

Re-verified all load-bearing claims in `wiki/experiment.md` against the live `gepa-ai/gepa` checkout (commit `92dadff`, v0.1.1) and the existing SAST application (`ai-sast-benchmark-A1b3/autoresearch/gepa/`). Appended a **"Validation against source (2026-07-03)"** section to `wiki/experiment.md`.

Headline corrections (GEPA evolved past the 2026-06-23 verification):
- `objective_scores` is no longer record-only: `frontier_type ∈ {instance, objective, hybrid, cartesian}` is consumed by `ParetoCandidateSelector`, and `optimize_anything` **defaults to `"hybrid"`** — per-objective champions natively survive as parents. The (β,V)-sweep-as-only-mitigation argument is weakened (still useful for frontier fill).
- Objectives are higher-is-better in frontier updates → cost must be fed as `neg_cost_usd`, not raw cost.
- `side_info["scores"]` → `objective_scores` automatically in `optimize_anything`; `AcceptanceCriterion` (pluggable) sees objective scores → hard cost gate needs no core patch; `on_candidate_rejected` callback → clean seam for the SkillOpt rejected-edit buffer.
- Reality check: the scanner is **single-stage** (one SKILL.md, one Claude session) — the 4-stage per-stage-model layout has no runner yet. Near-term candidate = one compound `{"sast_skill": prose + "[model: <id>]"}` with `run.py --model` (already supported; `meta.json` already carries `cost_usd`).

Deliverable: consolidated 8-task implementation plan (no GEPA core changes) at `ai-sast-benchmark-A1b3/docs/superpowers/plans/2026-07-03-multi-objective-gepa.md` — scoring (F_β + calibrated cost scalar + hard gate), `[model:]` directive parse/clamp, non-dominated (P,R,$) frontier readout CLI with cross-run union, reflection-prompt model/cost/bounded-edit context, cost-gate acceptance criterion, optimizer rewiring, rejected-edit buffer, smoke run + docs.

Follow-up check 2 (same day): per-stage models. Expected runner surgery dissolves — Claude Code subagents (`.claude/agents/*.md`, `model:` frontmatter = alias/full-id/inherit, loaded headless from cwd) carry per-stage heterogeneity natively; `run.py` already runs with cwd=workspace, allows the Agent tool, and persists per-model `modelUsage` to meta.json (free per-stage cost attribution for distinct models). Zero mandatory runner changes; real work = staged seed harness (must match monolithic baseline first) + multi-component candidate (`stage_<name>`, adapter-owned frontmatter) + per-stage ASI via `<component>_specific_info`. One empirical check open: subagent tokens in `modelUsage`. Added as Phase 2 (tasks 2a–2d) to the plan; experiment.md validation section updated.

Follow-up check (same day): bounded edits + negative-feedback memory vs GEPA's full-rewrite default. Findings appended to `wiki/experiment.md` validation section: full rewrite is structural in the default proposer (`output_extractor` swallows one fenced block as the whole component); true SkillOpt-style bounded ops need `custom_candidate_proposer` (clean seam, no core change; caveats: bypasses objective/background template, own-LM calls invisible to `max_reflection_cost`); core stores no rejected-candidate memory but the `on_proposal_start/end` + `on_candidate_rejected` callback triple reconstructs failed edits. Deferred design note (future Task 9) added to the plan.

## [2026-07-01] ingest | AlphaEvolve + ShinkaEvolve → `wiki/sources/alphaevolve.md`, `wiki/sources/shinkaevolve.md`

Two closely-related evolutionary-code-optimization sources ingested together as a pair.

**AlphaEvolve** (Google DeepMind, arXiv 2506.13131, Jun 2025):
- Evolutionary coding agent over **whole codebases** (not FunSearch-style single functions), Gemini Flash+Pro ensemble (breadth/depth split), one-or-more automated evaluators.
- Production wins inside Google: Borg scheduler heuristic recovering **0.7%** of global compute (in prod >1yr); Gemini training kernel **+23%** → **1%** overall training-time reduction; FlashAttention **+32.5%**; TPU Verilog simplification.
- Algorithmic: **4×4 complex matmul in 48 scalar multiplications** — first improvement over Strassen's 1969 algorithm in 56 years. Kissing number in 11D: new lower bound. On 50+ open math problems: rediscover SOTA 75%, improve SOTA 20%.
- Closed source. CORAL Stage-1 in the autonomy taxonomy.

**ShinkaEvolve** (Sakana AI, Robert Tjarko Lange et al., arXiv 2509.19349, Sep 2025):
- Open-source, sample-efficient counterpart. github.com/SakanaAI/ShinkaEvolve.
- Three named innovations:
  - Adaptive parent sampling (`weighted` / `power_law` / `beam_search` + archive-exploitation ratio)
  - **Code-novelty rejection sampling** — reject near-duplicate patches *before evaluation*, saving the scoring call
  - **Cost-aware UCB bandit** over LLM ensemble (Gemini 3-Flash/3.1-Pro, GPT-5-mini/GPT-5) — confidence radius inflated by per-call $ cost
- Islands + archive (fitness or crowding), three patch types (`diff` / `full` / `cross`), SLURM support.
- Signature result: **new SOTA circle packing in 150 samples**. Also AIME, ALE-Bench, MoE loss discovery.

Wiki updates:
- Two new source pages under `wiki/sources/` with cross-links to each other, AlphaEvolve↔ShinkaEvolve comparison tables.
- `wiki/index.md`: two rows in the Source Summaries table, two rows in the Source File Map, bumped last_updated.
- `wiki/concepts/evolutionary-optimization.md`: added two new sections (AlphaEvolve = whole-codebase evolution; ShinkaEvolve = sample-efficient open sibling); extended the "hierarchy of evolution" table with two rows (whole codebase; ensemble + cost-aware bandit); replaced the "FunSearch, AlphaEvolve" mention in the CORAL autonomy table with wiki links to the new pages; frontmatter sources/last_updated bumped.

Key positional claims:
- AlphaEvolve is the current high-water mark for the "LLM-as-mutation-operator" paradigm in terms of stakes (production deployment, hardware, algorithmic records).
- ShinkaEvolve is the **most reusable open substrate** in this family today — the closest thing to "AlphaEvolve you can actually run."
- ShinkaEvolve's cost-aware UCB bandit over an ensemble is a portable idea that resonates with the `wiki/experiment.md` "single-loop GEPA over [prompt, model]" framing — but at the *evolutionary-operator* layer rather than the *deployed-skill* layer. Worth remembering when the operator ensemble in an outer loop grows.

## [2026-06-25] decision | committed final solution = single-loop GEPA over [prompt, model]; outer-ordinal approach set aside → `wiki/experiment.md`

User chose the **inner, no-outer-loop** architecture and asked to discard the outer-ordinal additions from the prior entry. Added **"Final solution — single-loop GEPA over [prompt, model]"** and reconciled the doc so it no longer argues both sides.

Verified against `gepa-ai/gepa` source (`api.py`, `core/adapter.py`, `strategies/component_selector.py`): candidate is `dict[str, str]` (multi-component native), component selection is deterministic round-robin or all-at-once (not LLM-chosen). So model assignments can be optimized *inside* one GEPA run.

Committed design:
- **Compound component per stage** (`prompt + [model: …]` in one text blob) → a single round-robin mutation co-edits prose and model, avoiding prompt×model entanglement; enables per-stage model heterogeneity (opus-for-triage / haiku-for-extraction).
- **Extended ASI**: detection failures + cost attribution + per-stage model context (current model, realized cost, in-set cheaper/dearer alternatives) — defeats the "blind to model cost/capability" objection.
- **Global-dollar scalarization** (no per-model `c_ref`, since model is inner): `score_i = Q_i − cost_i/V` under `cost_i ≤ B_max`; calibration `V = B_max/0.3` (example V=$2.0/F-point).
- Validity enforcement (clamp+penalize out-of-set models); allowed set bounded by Evo's execution backends; objective-frontier readout carries per-stage model mix; only remaining outer loop is the optional `(β,V)` weight sweep.

Reconciliation edits (so the document is internally consistent):
- Marked the "model — GEPA cannot do this axis" subsection **superseded** (pointer to the final solution).
- **Removed** the outer-ordinal "Multi-stage model selection — the outer search becomes a vector" section (successive-weakening / quality-floor / `model_vector` / two-coupled-optimizers / AgentSpec model-ladder) — set aside per user decision.
- Rewrote "Net recommendation" and experimental-design step 5 from outer-ordinal `model_vector` sweep to the single-loop compound-component framing.

Note: the outer-ordinal analysis remains valid as an *alternative* (sample-efficient, exploits a capability↔cost ordering prior, deployment-oriented constrained min-cost) — it was removed here only because the user committed to the inner architecture, not because it was wrong. The prior log entry below records it.

## [2026-06-25] query | multi-stage model selection as an outer ordinal search → `wiki/experiment.md`

User has multi-stage skills where each stage uses a different model. Model is categorical (hard for GEPA's state-space search), and the combinatorial space is `M^S`. Key insight from user: capability roughly totally orders models (cost correlates), so the objective can be reframed as "find the weakest per-stage model that still meets a quality floor."

Updated `wiki/experiment.md`:
- New subsection **"Multi-stage model selection — the outer search becomes a vector"** inside the multi-objective extension, between "SkillOpt's role changes" and "The two-Pareto subtlety".
- Updated "Net recommendation" — `model_vector` swept via successive weakening with a quality-floor gate (single-stage degenerates to a categorical singleton); SkillOpt-style rejected-edit buffers applied to *both* skill prose and rejected model swaps as the coupling between inner/outer optimizers.
- Updated experimental design step 5 to call out the outer ordinal search and the verify-the-ordering precondition.
- Bumped `last_updated` to 2026-06-25.

Key conclusions:
- The reframe is a **problem-class change**: Pareto-search over `(P, R, cost, model_vector)` → constrained min-cost s.t. `quality ≥ floor`. Lossy by design (gives up upper-quality frontier regions); correct for deployment questions. Recover full surface by sweeping the floor.
- The capability ordering is a **prior, not a constraint** — three wrinkles to handle: partial-not-total ordering (verify all-weakest vs all-strongest per stage), stage interactions (multi-pass coordinate descent), end-to-end gating (no per-stage isolation).
- **Algorithm**: successive weakening — sweep stages weak-first by sensitivity, accept swap if quality ≥ floor. ~`S × M` evals per pass, ~2 passes. With S=4, M=4: ~32 evals vs 256 for full enumeration. Optional successive-halving on neighbors for noisy-eval handling.
- **Two coupled optimizers** alternate: outer = successive weakening on `model_vector`; inner = SkillOpt/GEPA on skill prose. SkillOpt's cross-model transfer is what makes the outer search exploitable; the typed rejected-swap buffer is the inner/outer coupling.
- **Evo mapping**: tree nodes = `(skill, model_vector)`; two edge types (skill edges = bounded prose edits; model edges = one-step weakening). `pareto_per_task` still handles CWE specialists at the frontier level; model-vector dimension lives in tree topology.
- AgentSpec extended with `Model search axis` declaring `pipeline_stages`, `model_ladder` per stage, the prior, and `quality_floor` as the load-bearing sweep knob.

## [2026-06-23] query | extending the vuln-detection experiment to multi-objective (precision, recall, cost, model) → `wiki/experiment.md`

User's real objective optimizes across (precision, recall, per-scan runtime cost, model), where model is a discrete categorical spanning closed- and open-weight models. Asked which methods fit, and whether GEPA handles it out of the box.

Updated `wiki/experiment.md`:
- Fixed the use-case table: `Model` is now a categorical optimization axis (not a frozen closed-weight base); added an `Objectives` row noting the multi-objective Pareto framing.
- Added a new section **"Multi-objective extension — optimizing across (precision, recall, cost, model)"**.
- Folded cost into the metric-design shape (objective vector `(P, R, $/scan)` + cost attribution) and the suggested experimental design (cost gate, model sweep, two-Pareto distinction).

Key conclusions:
- **GEPA** ([optimize-anything](sources/optimize-anything.md)) is the right primitive for the three numeric objectives — its design explicitly names cost as a Pareto objective — but NOT out of the box: you write the cost metric, must verify whether the shipped `gepa-ai/gepa` Pareto is over *objectives* vs *instances* (its known lineage is instance-Pareto, a diversity mechanism, not a `(P,R,cost)` frontier), and wire ASI for cost. A non-dominated filter over the objective vector is the likely ~small extension.
- **model is not GEPA's job** — it optimizes text artifacts, not categorical config. Sweep it as an outer categorical in Evo (whose non-Claude backends also run the open-weight models), then pool `(skill, model)` into one combined frontier.
- **SkillOpt's role flips** from generalization probe to enabler of the model axis (cross-model transfer = deploy on a cheaper open-weight model without a quality cliff).
- Demoted scalar-gated methods (auto-harness 80% gate, plain HonedHaiku) to control arms; reconsidered group-evolve/evoforge as genuine multi-objective shapes; AutoReason reframed as a cost lever.
- ~~Open action: spike `gepa-ai/gepa` to confirm whether `evaluate` is vector-valued.~~ **Resolved same day** (below).

### Follow-up — designed the cost-aware scalarization (added to `experiment.md`)

New subsection "Designing the cost-aware scalarization": instance = one repo scan (cost is per-instance natural, macro-over-CWE inside); quality = F_β from counts (avoids undefined per-instance P/R), β as recall/precision dial; cost normalized per-model to baseline `c_ref`, priced via willingness-to-pay `V`; `score_i = Q_i − ĉ_i/V` under a hard `B_max` gate; calibration recipe sets `V` so worst allowed cost penalty < 0.5 → "cheap junk wins" is structurally impossible. `(β, V)` are sweep axes → union `objective_scores` frontiers across runs. Raw `{P,R,cost}` → `objective_scores`; scalar → `scores`; cost attribution → ASI.

### Resolution — read the `gepa-ai/gepa` source (`core/adapter.py`, `core/state.py`)

- `EvaluationBatch.scores` is `list[float]` — **one scalar per evaluation instance**. `state.py`'s `_update_pareto_front_for_val_id` builds the frontier **per validation instance** (best on ≥1 instance). So GEPA's *search* is instance-Pareto + scalar acceptance — a diversity mechanism, NOT `(P,R,cost)` objective dominance.
- There **is** an `objective_scores` field (per-example `{objective → score}`) and an `_update_objective_pareto_front` method, but **no selection logic consumes it** — tracked/reported only.
- **Net:** no need to patch GEPA's frontier. Recipe = drive `scores` with a cost-aware scalarization, log raw `{P,R,cost}` to `objective_scores` (GEPA records the objective frontier for free), read it out at the end. Real limitation is exploration bias from the scalarization → mitigate by varying the cost weight across runs and unioning frontiers (Evo's outer sweep). Folded all of this into `experiment.md` (multi-objective section, metric design, experimental-design step 3, net recommendation).

## [2026-06-06] query | which methods fit a vuln-detection-via-Claude-Code-skills experiment? → `wiki/experiment.md`

User is running experiments on vulnerability-detection scanning workflows, implemented as Claude Code skills, against repos with ground-truth labels. Asked which wiki methods are most appropriate.

Filed the recommendation as a new top-level page `wiki/experiment.md` (linked from `index.md` under the Overview section).

Primary recommendation: **SkillOpt + Evo**, composed.
- SkillOpt because the optimization target is *literally* a skill document; bounded edits + held-out gate + rejected-edit negative buffer; strongest cross-model/cross-harness transfer evidence in the wiki.
- Evo because it is the only orchestrator that ships as Claude Code skills, auto-attaches a held-out-slice gate at `discover`, has a `pareto_per_task` frontier strategy that fits the multi-CWE objective, and treats gates as hard vetoes over score improvement.

Secondary fits: HonedHaiku (closest empirical analog — GEPA on Claude prompts for bug fixing; Goldilocks band caveat applies), GEPA/optimize_anything (the underlying primitive), RLM-GEPA's AgentSpec (adopt the idea even without the framework), auto-harness (single-thread baseline / control arm). AutoReason flagged as orthogonal but useful at *scan time* (finding-validation tournament).

Skipped with reasons: Agent0 (has ground truth, doesn't need bootstrap), AgentFlow/TRACE/SKILL-RL/SKILL-0 (closed-weight model rules out RL), ASI-Evolve (out of scope), CORAL/Group-Evolve/EvoForge (heavier than needed), AutoAgent variants (too broad — want to optimize skills not regenerate the agent), Meta-Harness (research artifact; SkillOpt is the productized successor), HALO (no live production traffic), Deep Research (HonedHaiku is the closer analog), EvoX (Evo's configurable frontier strategy covers most of this), Autogenesis (governance layer, not optimizer).

Key claim filed: **benchmark and metric design matter more than the optimizer choice.** The metric must return evidence-bounded per-failure descriptions (per RLM-GEPA's contract), not just F1 scalars; with that in place, every primary-fit system above composes cleanly.

Also filed a concrete experimental design: train/dev-gate/holdout/cross-CWE-holdout split, AgentSpec for scope declaration, `pareto_per_task` frontier with CWE as task axis, bounded edit budget as a tunable hyperparameter, ablations against argmax frontier and against the negative-signal buffer, plus cross-model transfer probe at the end.

## [2026-06-06] ingest | evo — autoresearch orchestrator with tree-search + RLM-inspired cross-cutting scans (evo-hq)

Source dropped: `sources/evo-hq` — single URL pointing to [github.com/evo-hq/evo](https://github.com/evo-hq/evo).

Evo (Alok Kumar Bishoyi, Apache-2.0) is one of the first packaged orchestrators for the Karpathy-style autoresearch loop. The `discover` skill explores the repo, picks what to measure, and instruments the benchmark; `optimize` runs the loop. Parallel subagents in isolated git worktrees hill-climb under a *tree-search* frontier (configurable: `argmax` / `top_k` / `epsilon_greedy` / `softmax` / `pareto_per_task`, the last credited to GEPA). Between rounds, *RLM-inspired cross-cutting scan subagents* read trace batches in parallel and surface gate-failure intersections + shared root causes. Gates are first-class primitives that *inherit down the experiment tree*; gate failure overrides score improvement. When `discover` builds a benchmark from scratch, it auto-attaches a held-out-slice score-floor gate. Eight execution backends (worktree/pool/ssh/modal/e2b/daytona/aws/azure) ship with the orchestrator.

Pages created:
- `wiki/sources/evo.md`

Pages updated:
- `wiki/index.md` — added row to source summaries and source file map; updated last_updated note to 2026-06-06
- `wiki/overview.md` — added row to "What can be optimized" table; added new loop architecture **Tree-structured parallel hill-climb** with frontier strategies + cross-cutting scans; added Evo to gating section (inheritable tree gates, hard veto, auto-attached held-out-slice gate); added Evo to feedback-signals section (cross-cutting scans as between-round phase); added three new open questions (tree vs. flat population, packaged autoresearch, discarded-hypothesis storage as standard)
- `CLAUDE.md` — added `evo-hq → wiki/sources/evo.md` slug entry
- `wiki/concepts/harness-optimization.md` — added Evo entry to "Approaches" + to the comparison table
- `wiki/concepts/evolutionary-optimization.md` — added Evo section with the full frontier-strategy table; added row to "Hierarchy of Evolution"
- `wiki/concepts/regression-gating.md` — added **Inheritable Tree Gates (Hard Veto)** as a new gating approach
- `wiki/concepts/self-improvement-loop.md` — added Evo's gating mode to the gate-phase list; added **Tree-structured parallel hill-climb with cross-cutting scans** as a new loop architecture
- `wiki/concepts/feedback-signals.md` — added two new named signal types: **Between-round cross-cutting scans** (Evo's RLM-inspired between-round phase, distinguished from ASI-Evolve's per-experiment Analyzer and HALO's production-traffic RLM by *standing phase of the loop* and *systemic-pattern targeting*) and **Discarded-hypothesis store** (negative-direction signal at experimental-direction granularity, parallel to SkillOpt's rejected-edit buffer at text-edit granularity)

Key analytical claims added:
- Evo's loop sits *between* single-thread hill-climb and flat population evolution: tree search preserves lineage and exposes the selection operator as a tunable knob, with `pareto_per_task` carrying GEPA's intuition (don't average specialists into mediocrity) into a tree-search context.
- Gate inheritance down the experiment tree is the safety analog of [Autogenesis](sources/autogenesis.md)'s versioned-resource lineage applied to *safety constraints* rather than artifacts. Combined with the *hard-veto-over-score* commitment, this is the strongest gating commitment in the wiki — stricter than auto-harness's 80% soft threshold and stricter than the implicit accept-if-better rule used by EvoForge / AutoAgent (KevinRGU).
- Auto-attached held-out-slice score-floor gate at `discover`-time is a notable UX choice: generalization protection becomes the default of the bootstrap step, not something the user has to remember to wire up. This is a quiet but important productization of the held-out-eval gating idea from [Meta-Harness](sources/meta-harness.md).
- Cross-cutting scans extend the trace-compression pattern of [HALO](sources/halo.md) (production-traffic) and [ASI-Evolve](sources/asi-evolve.md) (experimental output) to a third niche: *between-round trace batches inside a loop*. Crucially they target *gate-failure intersections* and *shared root causes across traces* — systemic patterns over branches, not per-experiment diagnosis. This makes feedback compression a standing phase of the loop rather than a one-shot analyzer.
- Evo's *discarded-hypothesis* bucket is unusual — most systems retain only successful branches. SkillOpt mined rejected text edits as negatives; Evo does this at the granularity of experimental directions. Plausibly the start of a pattern: negative-result storage as a standard piece of population/tree agentic search.
- Evo is the first wiki entry to package *multi-backend execution* (worktree/pool/ssh/modal/e2b/daytona/aws/azure) and a *dashboard* as part of the loop primitive, rather than leaving them as integration work.

No empirical results recorded — the repo is positioned as packaged infrastructure rather than a benchmark-driven research artifact (similar stance to [RLM-GEPA](sources/rlm-gepa.md) and [HALO](sources/halo.md)).

## [2026-05-30] ingest | rlm-gepa — GEPA-based optimizer for Recursive LM skill instructions (Trampoline AI)

Source dropped: `sources/rlm_gepa` (three URLs — repo, subdir, paywalled X post).

predict-rlm (Trampoline AI, MIT license) is a production-grade Recursive Language Model runtime built on the MIT CSAIL line of work (Alex L. Zhang, Tim Kraska, Omar Khattab). Root LM interacts with a programmatic REPL and issues sub-calls via DSPy signatures; runs in a WASM sandbox. The pitch: *"a single line can represent 1M sub-calls — in direct contrast to agents like Claude Code that must mechanically emit each sub-agent call one at a time"*; the root LM stays *"well within its comfortable operating range"* by avoiding long raw context.

`rlm_gepa` is the subproject that optimizes the *text components* of an RLM (typically skill instructions) using a GEPA-style proposer over scored execution traces.

Pages created:
- `wiki/sources/rlm-gepa.md`

Pages updated:
- `wiki/index.md` — added row to source summaries and source file map; updated last_updated note
- `wiki/overview.md` — added row to "What can be optimized" table; added to optimizer/target separation paragraph (now four systems); added to feedback-signals section with the evidence-bounded contract; added to knowledge-accumulation list; added two new open questions (AgentSpec as cross-literature standard; RLM as default scaffold)
- `CLAUDE.md` — added `rlm_gepa → wiki/sources/rlm-gepa.md` slug entry
- `wiki/concepts/harness-optimization.md` — added RLM-GEPA entry to "Approaches" list and to the comparison table
- `wiki/concepts/feedback-signals.md` — added two new named signal types: **Evidence-bounded feedback contract** ("name failures, don't prescribe rewrites" sharpened into a design constraint) and **Declared optimization context (AgentSpec)** (typed, declared brief on what's in-scope)
- `wiki/concepts/knowledge-accumulation.md` — added RLM-GEPA entry; noted the convergent finding with SkillOpt (independent groups arriving at "structured prose skill instructions as the natural unit of accumulation")

Key analytical claims added:
- The optimizer/target separation pattern now has four exemplars: HALO, AutoReason, SkillOpt, RLM-GEPA. RLM-GEPA's specific split is executor (runs RLM, collects traces) vs. proposer (reads traces, edits skill instructions).
- The "RLM" in HALO and the "RLM" in predict-rlm are the *same primitive* from the same MIT CSAIL line of work — not different things sharing a name. HALO uses an RLM specialized for trace analysis; predict-rlm provides the general runtime plus an optimization framework. The wiki should treat "RLM" as a substrate, not a system.
- The evidence-bounded feedback contract is a clean restatement of optimize_anything's ASI, sharpened into a design rule: the metric authors failure *descriptions*, the proposer authors *fixes*. Prescriptive metrics collapse the optimizer/target separation.
- AgentSpec is a new category of feedback — not "this run failed because X" but a standing typed brief on what kinds of behavioral changes count as in-scope. Most prior systems leave this implicit and force the optimizer to infer it from traces.

No empirical results recorded — the X post is paywalled (HTTP 402) and the repo is positioned as production infrastructure rather than a benchmark-driven artifact.

## [2026-05-26] lint | overview refresh — folded evoforge, honedhaiku, autoreason, halo, skillopt into wiki/overview.md

User flagged that the overview was stale (last_updated 2026-04-18) — five newer sources were already in the index but missing from the synthesis. Read all five source pages and integrated their distinctive contributions:

- Expanded the "What Can Be Optimized" table with 5 new rows (population of harnesses, system prompt only, per-query output, production-trace-driven harness, skill document as trainable state)
- Added two new loop architectures: *Specialist optimizer/target separation* (HALO + AutoReason + SkillOpt converge on this pattern) and *Inference-time per-query tournaments* (AutoReason)
- New section: **The Productive Band (Goldilocks Zone) for Prompt Optimization** — HonedHaiku and AutoReason independently observed a baseline-dependent productive range; SkillOpt's bounded edits ("textual learning rate") may widen this band
- Extended Knowledge Accumulation section with SkillOpt's transfer evidence (strongest in the wiki: +15.2% cross-model, +31.8% cross-harness)
- Added 10 new rows to the empirical results table (EvoForge, HonedHaiku, AutoReason, HALO ×2 models, SkillOpt ×4 settings)
- Added gating mechanisms from SkillOpt (rejected-edit buffer as negative mining) and AutoReason (Borda tournament as the gate)
- Added HALO's signal-compressor framing in the feedback-signals section (when feedback is too rich, a dedicated compressor becomes load-bearing)
- Added 4 new open questions covering optimizer/target separation, textual hyperparameters, what HALO does without production traffic, and composition of inference/deployment/training-time loops

Frontmatter bumped: sources list updated, last_updated → 2026-05-26.

---

## [2026-04-28] ingest | skillopt — skill document as the trainable state of a frozen LLM

Sources processed:
- `sources/skillOpt` → microsoft.github.io/SkillOpt + arXiv 2605.23904 (Microsoft Research) → `wiki/sources/skillopt.md`

Pages created:
- `wiki/sources/skillopt.md`

Pages updated:
- `wiki/concepts/knowledge-accumulation.md` — added "Skill Document as Trainable State" section; rejected-edit buffer as secondary persistent store; optimizer-side meta-skill as small meta-evolution layer; transfer evidence as strongest in the wiki
- `wiki/concepts/harness-optimization.md` — added SkillOpt entry (skill document with bounded structured edits); expanded comparison table
- `wiki/concepts/regression-gating.md` — added "Edit Budget (Textual Learning Rate)" section as prior-restraint primitive distinct from accept/reject gating
- `wiki/concepts/feedback-signals.md` — added "Rejected-edit buffer (negative signal)" — hard-negative mining analog
- `wiki/concepts/self-improvement-loop.md` — added "Skill-document gradient descent" loop architecture; explicitly frames the loop as SGD over a text artifact (document ↔ weights, optimizer model ↔ gradient, edit budget ↔ learning rate, validation gate ↔ accept/reject)
- `wiki/index.md` — new entry in source table and file map

Key findings:
- SkillOpt (Microsoft Research, arXiv 2605.23904): treats a single markdown skill document as the model's "trainable state"; frozen target model + separate optimizer model architecture echoes HALO and AutoReason. Best-or-tied-best on 52/52 model × benchmark combinations (7 models × 6 benchmarks); ALFWorld 70.9% → 85.8%
- **Edit budget as textual learning rate**: first explicit framing in the wiki of edit magnitude as a learning-rate hyperparameter — prior restraint on proposal size, distinct from post-hoc gating
- **Rejected-edit buffer**: rare in the wiki — most systems discard failed candidates; SkillOpt retains them as negative signal (hard-negative mining analog)
- **Transfer is the strongest evidence in the wiki**: cross-model +15.2%, cross-harness +31.8%, self-optimizer +10.4% — `best_skill.md` is genuinely model- and harness-agnostic. Argues that structured prose accumulation has portability properties that weight-based accumulation (SKILL-0, TRACE) cannot match
- The SGD-over-prose framing is conceptually clean: every primitive of gradient descent has a textual analog in SkillOpt

Cross-cutting observation: SkillOpt completes a triad of "skill-centric" systems in the wiki — SKILL-RL (external skillbank + RL), SKILL-0 (skills internalized into weights), WebXSkill (dual executable + NL skills), and now SkillOpt (single skill document as primary trainable state). The four together span the space of where accumulated skill knowledge can live: external dynamic store, weights, dual code/NL artifacts, or single structured prose document. SkillOpt occupies the lightest-weight end of that spectrum and shows the best transfer properties.

---

## [2026-04-25] ingest | autoreason + halo — tournament self-refinement and production-trace harness optimization

Sources processed:
- `sources/autoreason` → github.com/NousResearch/autoreason (SHL0MS · HERMES AGENT, Nous Research) → `wiki/sources/autoreason.md`
- `sources/halo` → github.com/context-labs/halo (Context Labs) → `wiki/sources/halo.md`

Pages created:
- `wiki/sources/autoreason.md`
- `wiki/sources/halo.md`

Pages updated:
- `wiki/concepts/regression-gating.md` — added Tournament Gating (Borda Vote) section as a fifth gating approach; explicitly framed as "gating as primary loop control" rather than as a safety check
- `wiki/concepts/feedback-signals.md` — added two new signal types: Cross-trace pattern compression (HALO RLM) and Peer judgment via blind tournament (AutoReason); the latter avoids the prompt-bias pathology by replacing critique with comparison
- `wiki/concepts/harness-optimization.md` — added HALO entry (production-trace methodology with separated RLM/coding-agent components); expanded comparison table
- `wiki/concepts/self-improvement-loop.md` — added two new loop architectures: per-query tournament refinement (AutoReason — within-query, not across-query) and production-trace deployment loop (HALO — explicit live-traffic design)
- `wiki/index.md` — new entries in source table and file map

Key findings:
- AutoReason (Nous Research): replaces single-agent critique-revise with a per-iteration tournament between incumbent (A), adversarial revision (B), and synthesis (AB), judged by fresh blind agents via Borda count. Explicitly diagnoses three failure modes of naive self-refinement: prompt bias (critics hallucinate flaws), scope creep (unbounded growth), lack of restraint (never choose no-change). Ablation shows both B and AB are necessary. Results: 77% vs 73% on CodeContests, 40% vs 31% over best-of-6 at matched compute, perfect sweep on Haiku 3.5 writing tasks. Diminishing returns above ~60% baseline accuracy — echoes HonedHaiku's Goldilocks band
- HALO (Context Labs): production-trace harness optimization formalized as a methodology. OpenTelemetry traces → specialized Recursive Language Model (RLM) for cross-trace pattern analysis → diagnostic report → coding agent edits harness → redeploy. Architectural innovation is the explicit separation between trace analysis (RLM) and harness editing (coding agent). AppWorld results: +15.8pp dev success for both Gemini 3 Flash and Sonnet 4.6 (same magnitude across very different model strengths suggests harness was the bottleneck for both); +10.7pp on test indicates generalization

Cross-cutting observations:
- AutoReason occupies a unique position in the wiki: it's the first system explicitly designed as a *per-query inference-time* loop, distinct from all prior systems which optimize *across queries* (harness, weights, prompts, populations). It generalizes the gating concept — the tournament *is* the loop, not a check on top of it
- HALO is the cleanest formalization yet of the auto-harness pattern (NeoSigma) — same overnight production-trace loop, but with separated, named components: RLM for analysis, coding agent for edits. The RLM-as-trace-compressor is conceptually adjacent to ASI-Evolve's Analyzer (compresses experimental output) and TRACE's contrastive trajectory analysis (compresses success/failure pairs into capability gaps) — three different specialized compressors for three different feedback regimes
- Both sources strengthen the "specialized component for diagnosis, separate from the editor" pattern: HALO's RLM, ASI-Evolve's Analyzer, AutoReason's judge panel, TRACE's diagnostic stage. The bare critique-revise pattern continues to look weak across the literature

---

## [2026-04-22] ingest | evoforge + honedhaiku — population harness evolution and GEPA prompt optimization for bug fixing

Sources processed:
- `sources/evoforge` → github.com/haizelabs/EvoForge (Leonard Tang, Haize Labs) → `wiki/sources/evoforge.md`
- `sources/honedhaiku` → tim.waldin.net/blog/2026-04-19-hone-haiku-20pp (Tim Waldin) → `wiki/sources/honedhaiku.md`

Pages created:
- `wiki/sources/evoforge.md`
- `wiki/sources/honedhaiku.md`

Pages updated:
- `wiki/concepts/harness-optimization.md` — added EvoForge (population-scale harness evolution, three-tier `evolve.md/program.md/agent.py` abstraction) and HonedHaiku (GEPA for system prompt, Goldilocks band insight); expanded comparison table with both entries
- `wiki/concepts/evolutionary-optimization.md` — added EvoForge section (population hill-climbing extension of AutoAgent KevinRGU) and GEPA-for-coding-prompts section (HonedHaiku); updated hierarchy table with EvoForge row; removed duplicate hierarchy table
- `wiki/index.md` — new entries in source table and file map

Key findings:
- EvoForge (Haize Labs): population harness evolution — extends the `program.md → agent.py` meta-agent loop to run across a population in parallel; `evolve.md` steers population strategy; knowledge synthesis after each generation enables cross-agent learning (weaker than CORAL but more structured than independent runs); 2× Codex CLI, 10× baseline on GPT-5-nano
- HonedHaiku (Tim Waldin): GEPA applied to system prompt optimization for Claude Haiku bug fixing; +19.7pp on unseen bugs (65% → 85%); identifies Goldilocks band constraint (prompt optimization only effective in 50–70% performance range); training diversity critical (3 challenges → overfit, 20 across 5 repos → generalizes); converges in 4 of 20 iterations; GEPA diagnosis pinpoints single failure mode (fixing first visible test only)

Cross-cutting observation: EvoForge fills a gap in the wiki — it's the only system that applies population-level parallelism specifically to harness evolution (others use populations for solution evolution or architecture search). HonedHaiku adds empirical constraint analysis to GEPA's applicability domain: the Goldilocks band is a practical limit on when prompt optimization is worth attempting.

---

## [2026-04-18] ingest | autogenesis + trace + webxskill — three new sources on protocols, capability-decomposition, and executable skills

Sources processed:
- `sources/autogenesis` → arXiv 2604.15034 (Wentao Zhang) → `wiki/sources/autogenesis.md`
- `sources/trace` → Stanford Scaling Intelligence Lab blog + arXiv 2604.05336 + github.com/ScalingIntelligence/TRACE → `wiki/sources/trace.md`
- `sources/webxskill` → arXiv 2604.13318 (Microsoft Research + UNC aiming-lab) + github.com/aiming-lab/WebXSkill (code placeholder) → `wiki/sources/webxskill.md`

Pages created:
- `wiki/sources/autogenesis.md`
- `wiki/sources/trace.md`
- `wiki/sources/webxskill.md`

Pages updated:
- `wiki/concepts/self-improvement-loop.md` — added capability-decomposed training (TRACE) and protocol-governed self-modification (AGP) loop types; added lineage+rollback gating; new proposal-target rows (LoRA adapters, executable skills, typed versioned resources); consolidated weight-optimization table (AgentFlow / SKILL-RL / SKILL-0 / TRACE)
- `wiki/concepts/feedback-signals.md` — added capability isolation (TRACE) as a third strategy for tractable sparse signals (alongside enrichment and credit assignment); three-axis comparison table
- `wiki/concepts/knowledge-accumulation.md` — added executable-skills dual representation (WebXSkill), per-capability LoRA adapters (TRACE), protocol-native lineage (AGP) as three distinct persistence forms
- `wiki/concepts/regression-gating.md` — added lineage + rollback gating (AGP) as a reversible alternative to irreversible accept/reject decisions
- `wiki/overview.md` — added new rows to optimization targets table and empirical results table; added Modular Decomposition section; expanded open questions; expanded regression-gating bullets
- `wiki/index.md` — new entries in source table and file map

Key findings:
- Autogenesis (arXiv 2604.15034): protocol-level specification (RSPL + SEPL) of self-modifying agents; typed versioned resources for prompts/tools/agents/environments/memory; proposal/assessment/commitment operators with auditable lineage + rollback — "git + CI for agent internals"; no published code as of submission
- TRACE (Stanford Scaling Intelligence Lab, arXiv 2604.05336): capability-decomposed training rather than end-to-end — contrast trajectories to diagnose gaps, synthesize per-capability environments, train one LoRA adapter per capability, route at inference; +14.1 pts on τ²-Bench, +7 perfect scores on ToolSandBox; 47.0% vs GRPO 37.8% at 5,120 rollouts
- WebXSkill (Microsoft + UNC aiming-lab, arXiv 2604.13318): executable skills with dual grounded/guided deployment; parameterized program + NL guidance in one artifact; URL-graph retrieval; +9.8 pts WebArena, +12.9 pts WebVoyager; same lab as Agent0

Cross-cutting observation: the three sources represent three orthogonal directions for self-improvement — AGP formalizes safety/reversibility at the protocol layer; TRACE decomposes the optimization problem into capability-isolated sub-problems; WebXSkill introduces dual-representation artifacts that bridge code and natural language. Together they expand the overview's optimization-target table from 9 to 14 rows and strengthen the case that weight-based accumulation is splitting into distinct modular (TRACE) vs. monolithic (AgentFlow, SKILL-0) variants.

---

## [2026-04-08] ingest | group-evolve + skill0 — group evolution and skill accumulation

Sources processed:
- `sources/group-evolve` → arXiv 2602.04837 + group-evolving-agents.github.io (UC Santa Barbara) → `wiki/sources/group-evolve.md`
- `sources/skill0` → arXiv 2602.08234 (SKILL-RL) + arXiv 2604.02268 (SKILL-0, Zhejiang Univ.) → `wiki/sources/skill-rl-skill0.md`

Pages created:
- `wiki/sources/group-evolve.md`
- `wiki/sources/skill-rl-skill0.md`

Pages updated:
- `wiki/concepts/evolutionary-optimization.md` — added GEA section (group-as-unit evolution with performance-novelty selection); added new row to hierarchy table (evolutionary unit level)
- `wiki/concepts/knowledge-accumulation.md` — added SkillBank (SKILL-RL, external hierarchical skill store co-evolving with RL policy) and Weight Internalization (SKILL-0, the only system in the wiki storing accumulated knowledge in model weights via curriculum RL)
- `wiki/index.md` — new entries in source table and file map

Key findings:
- GEA (UC Santa Barbara, arXiv 2602.04837): group-as-evolutionary-unit is a new level in the evolution hierarchy; performance-novelty dual-criterion selection prevents premature convergence; 71.0% SWE-bench Verified (+14.3pp over prior self-evolving SOTA), 88.3% Polyglot (+20pp), 3.6× faster framework bug repair
- SKILL-RL (arXiv 2602.08234): recursive skill-augmented RL; SkillBank co-evolves with policy; +15.3% over baseline RL on ALFWorld, WebShop, and 7 search tasks
- SKILL-0 (arXiv 2604.02268, Zhejiang Univ.): internalizes skills into model weights via progressive curriculum; zero-shot deployment with <0.5k tokens/step; +9.7% ALFWorld, +6.6% Search-QA over baseline RL; first system in the wiki to store accumulated knowledge in weights (others all use external stores)

---

## [2026-04-07] ingest | agentflow — Stanford on-policy RL for multi-turn agents (ICLR 2026)

Sources processed:
- `sources/agentflow` → arXiv 2510.05592 + agentflow.stanford.edu (Stanford/TAMU/UCSD/Lambda Labs) → `wiki/sources/agentflow.md`

Pages created:
- `wiki/sources/agentflow.md`

Pages updated:
- `wiki/concepts/self-improvement-loop.md` — added in-the-flow on-policy training loop type (Flow-GRPO); first weight-optimization loop in the wiki
- `wiki/concepts/feedback-signals.md` — added AgentFlow's credit assignment approach as challenge to the rich-feedback-wins thesis; new section on credit assignment as alternative to signal richness
- `wiki/index.md` — new entry in source table and file map

Key finding:
- AgentFlow (Stanford, ICLR 2026): only system in the wiki that optimizes **policy weights** rather than harness/prompts/code; trains the planner in-the-flow of live task execution via Flow-GRPO; achieves +14.9% knowledge search, +14.0% GAIA, +14.5% math with a 7B model; notable because it achieves strong results with only a binary scalar reward via trajectory-level credit assignment, challenging the dominant thesis that rich diagnostic feedback is required for effective self-improvement

---

## [2026-04-06] ingest | asi-evolve + coral + deep-research — three new arXiv sources

Sources processed:
- `sources/asi-evolve` → arXiv 2603.29640 (GAIR-NLP, March 2026) → `wiki/sources/asi-evolve.md`
- `sources/coral` → arXiv 2604.01658 + GitHub + project page (MIT/NUS/Stanford, April 2026) → `wiki/sources/coral.md`
- `sources/deep-research` → arXiv 2604.02988 (Zeta Alpha, April 2026) → `wiki/sources/deep-research.md`

Pages created:
- `wiki/sources/asi-evolve.md`
- `wiki/sources/coral.md`
- `wiki/sources/deep-research.md`
- `wiki/concepts/knowledge-accumulation.md` (new concept page — cross-cuts all three sources + auto-harness/meta-harness)

Pages updated:
- `wiki/concepts/self-improvement-loop.md` — added Learn-Design-Experiment-Analyze loop (ASI-Evolve), multi-agent co-evolution with shared memory (CORAL), heartbeat pivoting gate, novelty filtering gate; expanded compounding section to reference knowledge-accumulation
- `wiki/concepts/feedback-signals.md` — added Analyzer distillation (ASI-Evolve), qualitative shared notes (CORAL), textual gradient/loss (Deep Research); added section on feedback at different time horizons
- `wiki/concepts/evolutionary-optimization.md` — added ASI-Evolve database sampling policies, CORAL Stage 1/2/3 taxonomy and multi-agent co-evolution, GEPA for prompt optimization (Deep Research); expanded hierarchy table
- `wiki/overview.md` — added new rows to optimization targets table and empirical results table; added multi-agent co-evolution and Learn-Design-Experiment-Analyze to loop architectures; added knowledge accumulation section; expanded open questions
- `wiki/index.md` — new entries in source table, concepts table, and file map

Key findings:
- ASI-Evolve (GAIR-NLP): first system to target all three AI development axes (architecture + data + RL algorithm) simultaneously; Analyzer agent solves feedback compression for long-horizon research loops; discovered RL algorithms beat human-designed GRPO by +11.67 on AIME24
- CORAL (MIT/NUS/Stanford): autonomous Stage 2 multi-agent evolution; shared Attempts/Notes/Skills memory enables emergent copycatting and consensus behaviors; four-agent runs beat best-of-4 independent runs; new SOTA on Polyominoes (89.4 vs 87%) and Kernel Engineering (1,103 cycles)
- Deep Research (Zeta Alpha): GEPA prompt optimization starting from minimal prompts (0.705) beats TextGrad starting from expert prompts (0.672); demonstrates GEPA generality across code, architecture, and prompt spaces; references Darwin-Gödel Machine as theoretical target

---

## [2026-04-04] ingest | autoagent + autoagent2 — two new GitHub sources

Sources processed:
- `sources/autoagent` → `hkuds/autoagent` (NL-driven agent builder, HKUDS) → `wiki/sources/autoagent-hkuds.md`
- `sources/autoagent2` → `kevinrgu/autoagent` (meta-agent harness optimizer) → `wiki/sources/autoagent-kevinrgu.md`

Pages created:
- `wiki/sources/autoagent-hkuds.md`
- `wiki/sources/autoagent-kevinrgu.md`

Pages updated:
- `wiki/concepts/harness-optimization.md` — added both as new approaches; expanded comparison table
- `wiki/concepts/self-improvement-loop.md` — added autoagent-kevinrgu to proposal types table
- `wiki/overview.md` — added NL-via-dialogue row to optimization targets; added to empirical results table
- `wiki/index.md` — new entries in source table and file map

Key findings:
- hkuds/autoagent: natural language as the complete harness-specification interface; human-in-the-loop (NL direction), not fully autonomous; GAIA ≈ Claude 3.5
- kevinrgu/autoagent: fully autonomous meta-agent hill-climbing; novel `program.md`/`agent.py` abstraction boundary; no published benchmarks

---

## [2026-04-04] ingest | Initial bulk ingest — 7 source files (8 URLs)

Processed all sources in `sources/` directory. Sources:
- `agent0` → Agent0 paper (arXiv 2511.16043)
- `auto-harness` → NeoSigma auto-harness repo (GitHub) + AutoHarness paper (arXiv 2603.03329)
- `autoresearch-vs-hyperparams` → Weco AI blog post
- `meta-harness` → Stanford IRIS Meta-Harness page
- `optimize-anything` → GEPA/optimize_anything blog (UC Berkeley + Stanford)
- `self-improving` → NeoSigma AI blog post (combined with auto-harness source page)
- `skydicover` → EvoX blog post (UC Berkeley Sky Lab)

Pages created:
- `wiki/overview.md`
- `wiki/sources/agent0.md`
- `wiki/sources/auto-harness.md` (covers both NeoSigma repo and blog)
- `wiki/sources/autoharness-arxiv.md`
- `wiki/sources/autoresearch-vs-hpo.md`
- `wiki/sources/meta-harness.md`
- `wiki/sources/optimize-anything.md`
- `wiki/sources/evox.md`
- `wiki/concepts/self-improvement-loop.md`
- `wiki/concepts/feedback-signals.md`
- `wiki/concepts/harness-optimization.md`
- `wiki/concepts/regression-gating.md`
- `wiki/concepts/evolutionary-optimization.md`
- `wiki/index.md`
- `CLAUDE.md` (schema)

# Wiki Log

Append-only record of ingests, queries, and lint passes.
Format: `## [YYYY-MM-DD] type | description`

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
- Open action before committing: spike `gepa-ai/gepa` to confirm whether `evaluate` is vector-valued.

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

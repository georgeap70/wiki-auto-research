---
title: "WikiSkill: Compiling Agent Experience into Persistent Knowledge for Skill Evolution"
type: source
tags: [knowledge-accumulation, skill-evolution, persistent-knowledge, three-layer-architecture, regression-gating, transfer, llm-wiki, context-engineering]
sources: [wiki.skill]
url: https://arxiv.org/abs/2608.27454
code: none
authors: Liyan Tang, Cyrus Rashtchian, Chun-Sung Ferng, Andrew Tomkins, Da-Cheng Juan, Tu Vu
affiliations: Google Research, Virginia Tech
arxiv: 2608.27454
last_updated: 2026-09-08
---

# WikiSkill (arXiv 2608.27454)

## Summary

WikiSkill adds a **persistent knowledge layer between raw experience and executable skills**, and shows that this single architectural change is worth more than the skill-evolution method wrapped around it. Its ablation is the cleanest result: giving the Skill Proposer access to an accumulated wiki lifts average benchmark performance from **48.7% to 63.7% (+15.0)** — larger than the gap between any two competing skill-evolution methods in the paper.

The framing is explicitly borrowed from **Karpathy's "LLM Wiki"** proposal: compile experience into persistent, compounding knowledge rather than leaving insights "scattered across optimization histories." Its diagnosis of prior work — [SkillOpt](skillopt.md), EvoSkill, Trace2Skill — is that they all analyze traces and update skills, but **none maintains what has been learned as a separate, evolving knowledge representation.**

This wiki is itself an instance of the pattern the paper argues for, which makes it a usefully self-referential source.

## Three-Layer Architecture

The agent workspace is split by **mutability and lifetime**, which is the load-bearing design decision:

| Layer | Path | Contents | Lifecycle |
|-------|------|----------|-----------|
| **Raw** | `raw/` | Complete step-by-step execution traces: reasoning, tool calls, outputs, final answers | **Permanent, write-once** (immutable) |
| **Wiki** | `wiki/` | `patterns/*.md` (failure modes / successful strategies + workarounds), `index.md` catalog, `logs.md` evolution log, `skill-impact.md` audit trail | **Compounding, never reset** |
| **Skills** | `skills/` | Active skill set: `SKILL.md` (full content) + `PURPOSE.md` (maps the skill back to the wiki patterns that motivated it) | **Reversible, conditional update** |

The asymmetry is the mechanism: **skills roll back, the wiki never does.** A rejected proposal still leaves permanent knowledge behind, so a failed iteration is not a wasted iteration.

`skill-impact.md` is written **programmatically by the outer harness**, not by an LLM — recording proposal metadata, target skill name, unified diff, validation score, and accept/reject outcome. It is an objective, ground-truth audit trail the Skill Proposer consults to avoid re-proposing rejected interventions. This is the same "make the record tamper-proof by keeping it out of the agent's hands" instinct as [HarnessDev](harnessdev.md)'s unassertable scoring and [Autogenesis](autogenesis.md)'s protocol-level lineage.

## The Loop

Four components per iteration, over state `(S_k, W_k)`:

1. **Inference Agent** — runs rollouts on the training split with the active skills injected in full into its system prompt, producing immutable traces. **Deliberately denied wiki access** (see ablation).
2. **Wiki Maintainer** `W'_k ← M_WM(W_{k-1}, T_sample,k)` — receives the full wiki plus a stratified sample of successful and failing traces. Performs **root-cause analysis** on failures and extracts successful strategies. Creates new pattern pages, updates existing ones via **incremental patch-based editing** (append / replace / insert spans), revises `index.md`, and appends findings to `logs.md`. No hard limit on patterns created or edited per iteration.
3. **Skill Proposer** `P_k ← M_P(W'_k, S_{k-1}, T_train,k)` — operates in **multi-turn ReAct style**. Rather than being handed a fixed pre-sampled trace set, it starts with only the wiki index, `skill-impact.md`, and a concise pass/fail summary, then **actively uses `read_file` to pull specific pattern pages and raw traces on demand.** This is on-demand retrieval as a context-budget strategy. Output is an **atomic proposal targeting a single skill** — either a creation or an incremental patch.
4. **Gating and Rollback** — evaluate candidate `S'_k` on the validation split; accept iff `R(T_val,k) > R_best`. `R_best` initializes to the **empty-skill baseline** validation score. Early termination if validation hits 1.0. On rejection, revert skills — **but never the wiki**.

## Results

Five benchmarks — LiveMathematicianBench (math), SealQA (web search), SpreadsheetBench (spreadsheets), OfficeQA (long-context doc QA), ALFWorld (embodied) — and five models. Baselines are three dedicated skill-evolution frameworks (**Trace2Skill, EvoSkill, [SkillOpt](skillopt.md)**) plus no-skill. All methods start from an **empty skill set**. Three independent full-evolution runs per method; significance by paired bootstrap (1,000 iterations, p < 0.05).

General prompt optimizers like [GEPA](optimize-anything.md) are deliberately excluded, citing SkillOpt's finding that specialized skill-evolution pipelines consistently beat general prompt optimization.

### Average across the five benchmarks

| Model | No skill | Trace2Skill | EvoSkill | SkillOpt | **WikiSkill** | vs. best competitor |
|-------|----------|-------------|----------|----------|---------------|---------------------|
| Qwen-3.5-4B | 26.2 | 32.1 | 33.7 | 35.2 | **38.5** | +3.3 |
| Qwen-3.5-9B | 29.9 | 36.7 | 42.3 | 40.2 | **47.4** | +5.1 |
| Qwen-3.6-27B | 39.4 | 47.3 | 53.3 | 50.7 | **63.3** | +10.0 |
| Gemma-4-31B | 41.3 | 45.8 | 43.4 | 49.1 | **54.9** | +5.8 |
| Gemini-3.5-Flash | 49.5 | 55.6 | 56.1 | 55.9 | **68.1** | +12.0 |

Individual highlights: Gemini LiveMath **33.0 → 72.6**, SpreadSheet **50.5 → 76.6**; Qwen-27B SpreadSheet **40.8 → 81.7** (+40.9), ALFWorld **52.8 → 77.6**.

Competing methods are notably **less reliable**, not just weaker: EvoSkill lifts Qwen-9B on LiveMath (28.2 → 58.1) but *degrades* Gemma-4-31B on the same benchmark (33.9 → 29.8); SkillOpt degrades Gemini on SealQA (29.4 → 28.2).

### Skill evolution complements model scaling

Within the Qwen family, gains over no-skill **increase with scale**: **+12.3 (4B), +17.5 (9B), +23.9 (27B)** — and on SpreadSheet, +6.5 / +9.3 / **+40.9**. Simultaneously, skills substitute for scale: **Qwen-3.5-9B + WikiSkill (47.4%) beats Qwen-3.6-27B with no skills (39.4%)**.

This cuts against the [Goldilocks band](honedhaiku.md) hypothesis, which held that text-only optimization pays off in a middling 50–70% baseline range and saturates above it. WikiSkill's advantage becomes *more* pronounced for stronger models. The reconciliation is likely the same one [SkillOpt](skillopt.md) suggested — structured artifacts widen the productive band — with an addition: stronger models are better at *both* halves of the loop, discovering better patterns and executing more elaborate procedures.

### Cross-model transfer: discovery and execution are different capabilities

| Inference model | Benchmark | No skill | Self-evolved | Best transferred |
|-----------------|-----------|----------|--------------|------------------|
| Qwen-3.5-9B | ALFWorld | 34.7 | 63.4 | **70.2** (from Qwen-27B) |
| Qwen-3.5-9B | SpreadSheet | 24.3 | 33.6 | **50.5** (from Qwen-27B) |
| Gemma-4-31B | LiveMath | 33.9 | 56.7 | **73.7** (from Qwen-27B) / 73.1 (from Qwen-**4B**) |
| Gemini-3.5-Flash | LiveMath | 33.0 | 72.6 | 73.9 (from Qwen-27B) |

**Transferred skills frequently beat self-evolved skills**, and transfer works *upward* — Qwen-3.5-4B's skills lift Gemma-4-31B to 73.1 on LiveMath and 66.9 on ALFWorld. The sharpest case: **Qwen-3.5-4B's OfficeQA skills hurt Qwen-3.5-4B (30.2 → 28.5) but help Qwen-3.6-27B (42.1 → 52.9)** — a model authored procedural knowledge more useful to another model than to itself.

The paper's conclusion is a distinction the wiki has been missing: **self-evolution conflates *discovering* useful procedural knowledge with *executing* it at inference time.** These are separate capabilities, and a system can be good at one and bad at the other. It is the same separation [HarnessDev](harnessdev.md) enforces architecturally with its creator/executor split.

### Negative transfer has an identified mechanism

**Qwen-3.5-4B's SpreadSheet skills crater Gemini from 50.5 → 18.1**, while Qwen-27B's improve it to 63.4. Error analysis names two causes:

1. The 4B model's skills encode **low-level workarounds** (single-line Python commands, string-conversion rules) that help a small model avoid execution failures but **constrain a stronger model** from writing comprehensive end-to-end scripts.
2. **Fragmented diagnostic procedures** introduce redundant tool calls that exhaust Gemini's interaction budget before completion.

So transferability depends on whether a skill captures a **general procedure** or a **model-specific workaround** — a clean refinement of the wiki's [model-specificity debate](../concepts/harness-optimization.md), and a mechanism for why [Self-Harness](self-harness.md)'s model-specific edits and [evolve-the-harness](evolve-the-harness.md)'s transferable code mechanisms can both be right. Weaker models generate more workarounds; those workarounds are precisely what fails to transfer.

### The ablation: where persistence actually pays

Gemini-3.5-Flash, four configurations varying wiki access. (When the Proposer has no wiki access, the Wiki Maintainer is removed too — eliminating persistent accumulation entirely.)

| Inference Agent wiki | Skill Proposer wiki | LiveMath | SealQA | SpreadSheet | OfficeQA | Avg |
|:---:|:---:|---|---|---|---|---|
| *no skill baseline* | | 33.0 | 29.4 | 50.5 | 48.6 | 40.4 |
| ✓ | ✗ | 43.8 | 42.0 | 44.4 | 51.0 | 45.3 |
| ✗ | ✗ | 51.3 | 38.4 | 49.9 | 55.2 | 48.7 |
| ✓ | ✓ | 64.8 | 42.8 | 80.2 | 55.6 | 60.9 |
| **✗** | **✓** | **72.6** | **44.7** | 76.6 | **60.7** | **63.7** |

Two findings:

- **Persistent knowledge for the Proposer is the dominant term: 48.7 → 63.7 (+15.0)**, with LiveMath 51.3 → 72.6 and SpreadSheet 49.9 → 76.6. Without accumulation across iterations the Proposer cannot resolve intricate failure modes.
- **Wiki access for the Inference Agent *hurts*: 63.7 → 60.9** (LiveMath 72.6 → 64.8). The hypothesis: when the Inference Agent can read both skills and wiki during training rollouts, it solves tasks using knowledge from the wiki rather than the skills, making the resulting **trajectories less informative about skill quality**. Contaminating the rollout with the diagnostic layer degrades the signal the diagnostic layer depends on.

That second result is subtle and generalizable — a caution for any loop where the improved artifact and the accumulated knowledge base are both readable at execution time.

### Artifact statistics and refinement dynamics

Qwen models produce long skills (118.9–128.6 lines); Gemma-4-31B (45.1) and Gemini (81.2) are far more compact — and Gemma/Gemini score higher per line. Per model, 6.3–8.9 patterns are created and 7.0–18.4 edited. By benchmark, SpreadSheet yields the longest skills (142.5 lines) and most patterns (9.8); LiveMath the shortest (84.6) and fewest (4.4). **All wiki pattern creations and edits are retained.**

Accepted skill updates are **not front-loaded**: iterations 0–1 account for only 39–52%, with substantial fractions continuing into middle (2–4) and late (5–7) stages — on SealQA, 33% middle and 28% late. Refinement continues throughout, which is what accumulation is supposed to enable.

### Case study (ALFWorld, Qwen-3.6-27B)

Iteration 0: the Wiki Maintainer documents a `take-examine-move-loop` pattern; the Proposer's `goal-directed-action` skill fails validation and is **rejected** — but the diff and outcome persist in `skill-impact.md`. Iteration 1: informed by that audit trail, the Proposer creates `break-repetition-loop` with a **concrete** action rule ("Never Return an Item to Its Origin Location"), which is accepted. Its `PURPOSE.md` records *why*: the previous attempt was rejected for being too abstract. As new loop variants appear, the Maintainer accumulates `multi-operation-loop.md`; at iteration 4 the Proposer refines the skill with a further rule ("Each Operation Type ONCE Per Item").

The mechanism on display: the rejection was informative, and only survived because the wiki isn't rolled back.

## What Is Optimized / Feedback / Gating

| Axis | WikiSkill |
|------|-----------|
| Optimization target | A set of filesystem skills (`SKILL.md` + `PURPOSE.md`), co-evolved with a persistent wiki. No weights; harness held fixed |
| Feedback signal | Sampled successful + failing execution traces, root-cause-analyzed into pattern pages; plus a programmatic accept/reject audit trail |
| Loop structure | Four-component sequential loop (Inference → Wiki Maintainer → Skill Proposer → Gate), with an asymmetric two-speed state: reversible skills, irreversible knowledge |
| Gating | Strict improvement over the running best validation score; rollback of skills only; early stop at 1.0 |
| Human involvement | None after setup (benchmarks, splits, budget) |

## Why It Matters

- **It isolates persistence as the active ingredient.** The wiki has long asserted that [knowledge accumulation](../concepts/knowledge-accumulation.md) is what makes loops compound; this is the first source to **ablate it directly within one system** and put a number on it (+15.0). The result also implies that much of the reported difference between skill-evolution *methods* may be a proxy for how well each retains cross-iteration knowledge.
- **The reversible/irreversible split is a genuinely new primitive.** [SkillOpt](skillopt.md)'s rejected-edit buffer and [Evo](evo.md)'s discarded-hypothesis store keep negative signal; [Autogenesis](autogenesis.md) makes everything rollback-able. WikiSkill's contribution is running **two layers at different speeds on purpose** — a fast reversible artifact under a gate, over a slow irreversible knowledge base immune to it. It answers the awkward question of what a failed iteration is *for*.
- **Retrieval-on-demand instead of pre-sampling.** The ReAct-style Proposer pulling specific pattern pages and traces via `read_file` is a different answer to the context-budget problem than [HALO](halo.md)'s or [ASI-Evolve](asi-evolve.md)'s dedicated compressor agents: let the consumer decide what it needs, and keep an index. As accumulated knowledge grows past any context window, this is likely the more scalable pattern.
- **Contamination of the rollout degrades the diagnostic signal.** The finding that Inference-Agent wiki access *hurts* is a specific, non-obvious design constraint with wide applicability.
- **It is the mechanism for the transfer debate.** "General procedure vs. model-specific workaround" explains negative transfer concretely, and predicts that skills evolved by weak models will transfer worst — which the data shows.

## Limitations (the paper's own)

- **Skill retrieval is not evaluated.** Active skills are injected in full into the system prompt, deliberately eliminating triggering/retrieval failures as a confound. This becomes a real problem as skill libraries grow — the paper points to a concurrent skill-retrieval benchmark (SkillRet) as the missing piece.
- **The strict gate excludes neutral proposals** that preserve immediate performance but could enable later gains. Adopted to match prior frameworks for fair comparison; more flexible acceptance criteria are flagged as future work. (Contrast [Self-Harness](self-harness.md)'s *non-detrimental* rule, which admits neutral edits.)
- **No wiki pruning.** Pattern pages, logs, and diffs accumulate without any automated pruning mechanism — the "forgetting" open question this wiki's [knowledge-accumulation](../concepts/knowledge-accumulation.md) page has flagged, now stated as an acknowledged gap by a system that hit it. [SKILL.state](skill-state.md) is the aggressive opposite position on the same axis.
- **No very-long-horizon tasks** (hundreds of actions or multiple hours), and no **online** skill adaptation within a single rollout.

## Connections

- [concepts/knowledge-accumulation](../concepts/knowledge-accumulation.md) — the anchor source: persistence ablated and quantified; the two-speed reversible/irreversible pattern
- [concepts/regression-gating](../concepts/regression-gating.md) — strict-improvement gate with layer-selective rollback; neutral-proposal exclusion as an acknowledged cost
- [concepts/self-improvement-loop](../concepts/self-improvement-loop.md) — four-component loop; rollout contamination as a loop-design constraint
- [concepts/feedback-signals](../concepts/feedback-signals.md) — root-cause pattern pages + a programmatic accept/reject audit trail; on-demand retrieval over pre-sampling
- [concepts/context-engineering](../concepts/context-engineering.md) — patch-based pattern editing, the same anti-erosion discipline as ACE
- [concepts/evaluating-self-improvement](../concepts/evaluating-self-improvement.md) — the discovery/execution capability split
- [sources/skillopt](skillopt.md) — the closest prior system and a direct baseline it beats on all five models
- [sources/skill-state](skill-state.md) — the opposite pole on retention: accumulate across runs (WikiSkill) vs. discard within one (SKILL.state)
- [sources/harnessdev](harnessdev.md) — independently arrives at the creator/executor (discovery/execution) separation
- [sources/ace](ace.md) — itemized delta updates against context collapse; WikiSkill's pattern pages are the same discipline at page granularity
- [sources/interaction-trajectory-mining](interaction-trajectory-mining.md) — the negative case for offline skill mining; WikiSkill's live-gated curation is what it lacked

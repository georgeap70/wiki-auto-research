# Self-Improving Agentic Systems — Wiki Index

A catalog of all pages. Read this first when answering queries. Last updated: 2026-06-06 (+ evo).

---

## Overview

| Page | Summary |
|------|---------|
| [Overview](overview.md) | High-level synthesis of the field: the core loop, what can be optimized, key findings, open questions |
| [Experiment](experiment.md) | Applying the wiki to a specific use case: optimizing Claude Code skills for vulnerability detection against ground-truth-labeled repos; primary recommendation = SkillOpt + Evo |

---

## Source Summaries

One page per raw source. See `sources/` directory for raw URLs.

| Page | Raw source | Code | Summary |
|------|-----------|------|---------|
| [Agent0](sources/agent0.md) | `sources/agent0` | [github](https://github.com/aiming-lab/Agent0) | Two-agent co-evolution bootstraps complex tool use from zero data (arXiv 2511.16043) |
| [auto-harness (NeoSigma)](sources/auto-harness.md) | `sources/auto-harness`, `sources/self-improving` | [github](https://github.com/neosigmaai/auto-harness) | Agent iterates its own harness overnight; +39% on Tau3 with no model changes |
| [AutoHarness (paper)](sources/autoharness-arxiv.md) | `sources/auto-harness` | none | Google DeepMind: agent synthesizes constraint code from environmental feedback (arXiv 2603.03329) |
| [AutoResearch vs HPO](sources/autoresearch-vs-hpo.md) | `sources/autoresearch-vs-hyperparams` | [weco-cli](https://github.com/WecoAI/weco-cli), [autoresearch](https://github.com/karpathy/autoresearch) | Claude Code agent beats Optuna TPE by operating over full code space |
| [Meta-Harness](sources/meta-harness.md) | `sources/meta-harness` | [github](https://github.com/stanford-iris-lab/meta-harness-tbench2-artifact) | Stanford: full harness optimization with rich execution trace context |
| [optimize_anything](sources/optimize-anything.md) | `sources/optimize-anything` | [github](https://github.com/gepa-ai/gepa) | UC Berkeley/Stanford: universal text-optimization API with ASI + Pareto search |
| [EvoX](sources/evox.md) | `sources/skydicover` | [github](https://github.com/skydiscover-ai/skydiscover) | UC Berkeley: meta-evolution — the search strategy itself is evolvable |
| [AutoAgent (HKUDS)](sources/autoagent-hkuds.md) | `sources/autoagent` | [github](https://github.com/hkuds/autoagent) | HKUDS: NL-driven agent builder with self-developing architecture; GAIA ≈ Claude 3.5 |
| [AutoAgent (KevinRGU)](sources/autoagent-kevinrgu.md) | `sources/autoagent2` | [github](https://github.com/kevinrgu/autoagent) | Meta-agent hill-climbing loop; program.md directive → auto-edited agent.py harness |
| [ASI-Evolve](sources/asi-evolve.md) | `sources/asi-evolve` | [github](https://github.com/GAIR-NLP/ASI-Evolve) | GAIR-NLP: Learn–Design–Experiment–Analyze; AI discovers new neural architectures, data pipelines, and RL algorithms better than human-designed baselines (arXiv 2603.29640) |
| [CORAL](sources/coral.md) | `sources/coral` | [github](https://github.com/Human-Agent-Society/CORAL) | MIT/NUS/Stanford: autonomous multi-agent co-evolution with shared Attempts/Notes/Skills memory; emergent copycatting and consensus behaviors; new SOTA on Polyominoes and Kernel Engineering (arXiv 2604.01658) |
| [Deep Research](sources/deep-research.md) | `sources/deep-research` | none | Zeta Alpha: TextGrad + GEPA prompt optimization for four-agent deep research pipeline; GEPA from minimal start beats expert-designed prompts (arXiv 2604.02988) |
| [AgentFlow](sources/agentflow.md) | `sources/agentflow` | none | Stanford/TAMU/UCSD/Lambda: in-the-flow on-policy RL (Flow-GRPO) trains planner weights during live execution; +14.9% knowledge search, +14.0% GAIA with 7B model; ICLR 2026 oral (arXiv 2510.05592) |
| [Group-Evolving Agents](sources/group-evolve.md) | `sources/group-evolve` | [project](https://group-evolving-agents.github.io/) | UC Santa Barbara: group-as-evolutionary-unit with shared experience pool; performance-novelty selection; SWE-bench 71% vs 56.7% SOTA (arXiv 2602.04837) |
| [SKILL-RL + SKILL-0](sources/skill-rl-skill0.md) | `sources/skill0` | none | SkillRL: recursive skill-augmented RL with co-evolving SkillBank (+15.3% on ALFWorld/WebShop); SKILL-0: curriculum RL internalizes skills into weights for zero-shot deployment (arXiv 2602.08234, 2604.02268) |
| [Autogenesis](sources/autogenesis.md) | `sources/autogenesis` | none | Wentao Zhang: protocol (AGP) for self-modifying agents with versioned resources, proposal/assessment/commitment operators, auditable lineage + rollback — "git + CI for agent internals" (arXiv 2604.15034) |
| [TRACE](sources/trace.md) | `sources/trace` | [github](https://github.com/ScalingIntelligence/TRACE) | Stanford Scaling Intelligence: capability-targeted training — contrast failed/successful trajectories → synthesize per-capability environments → one LoRA adapter per capability → router at inference; +14.1 on τ²-Bench |
| [WebXSkill](sources/webxskill.md) | `sources/webxskill` | [github](https://github.com/aiming-lab/WebXSkill) | Microsoft+UNC (aiming-lab): executable skills for web agents — parameterized program + NL guidance, dual grounded/guided deployment, URL-graph retrieval; +9.8 pts WebArena, +12.9 pts WebVoyager (arXiv 2604.13318) |
| [EvoForge](sources/evoforge.md) | `sources/evoforge` | [github](https://github.com/haizelabs/EvoForge) | Haize Labs: population of agent harnesses evolving in parallel; `evolve.md → program.md → agent.py` three-tier abstraction; 2× Codex CLI, 10× baseline on GPT-5-nano |
| [HonedHaiku](sources/honedhaiku.md) | `sources/honedhaiku` | none | Tim Waldin: GEPA optimizes Claude Haiku system prompt for bug fixing; +19.7pp on unseen bugs (65% → 85%); Goldilocks band insight; converges in 4 iterations |
| [AutoReason](sources/autoreason.md) | `sources/autoreason` | [github](https://github.com/NousResearch/autoreason) | Nous Research: tournament-based self-refinement (incumbent vs. adversarial vs. synthesis, Borda voting); fixes prompt bias, scope creep, lack-of-restraint in critique-revise loops; +4pp on CodeContests, 40% vs 31% over best-of-6 |
| [HALO](sources/halo.md) | `sources/halo` | [github](https://github.com/context-labs/halo) | Context Labs: production OpenTelemetry traces → specialized RLM engine → coding agent edits harness → redeploy; +15.8pp dev / +10.7pp test on AppWorld for both Gemini 3 Flash and Sonnet 4.6 |
| [SkillOpt](sources/skillopt.md) | `sources/skillOpt` | [project](https://microsoft.github.io/SkillOpt/) | Microsoft Research: skill document as "trainable state" of a frozen LLM; rollout → reflection → bounded edits (textual learning rate) → validation; best-or-tied-best on 52/52 settings (7 models × 6 benchmarks); strongest cross-model (+15.2%) and cross-harness (+31.8%) transfer in the wiki (arXiv 2605.23904) |
| [RLM-GEPA](sources/rlm-gepa.md) | `sources/rlm_gepa` | [github](https://github.com/Trampoline-AI/predict-rlm) | Trampoline AI: GEPA-based optimizer for Recursive LM skill instructions; `AgentSpec` declares what's in-scope so the optimizer doesn't have to infer project context; evidence-bounded feedback contract (name failures, don't prescribe rewrites); production-grade infra built on MIT CSAIL RLM line of work |
| [Evo](sources/evo.md) | `sources/evo-hq` | [github](https://github.com/evo-hq/evo) | evo-hq (Alok Kumar Bishoyi): autoresearch orchestrator; `discover` instruments the benchmark, parallel subagents in isolated worktrees hill-climb under tree-search frontier strategies (`argmax`/`top_k`/`epsilon_greedy`/`softmax`/`pareto_per_task`); RLM-inspired cross-cutting scans between rounds; gates inherit down the experiment tree; held-out-slice score floor auto-attached on discover; 8 execution backends |

---

## Concepts

| Page | Summary |
|------|---------|
| [Self-Improvement Loop](concepts/self-improvement-loop.md) | The core measure→fail→propose→gate cycle; loop architectures (single, two-agent, population, meta) |
| [Feedback Signals](concepts/feedback-signals.md) | Scalar vs. rich diagnostic feedback; why rich traces produce better proposals; ASI |
| [Harness Optimization](concepts/harness-optimization.md) | Optimizing the wrapper around a base LLM (prompts, tools, constraints) without touching weights |
| [Regression Gating](concepts/regression-gating.md) | How safe self-improvement is enforced: threshold, Pareto, held-out eval, stagnation-based |
| [Evolutionary Optimization](concepts/evolutionary-optimization.md) | Population-based search; LLMs as operators; GEPA; meta-evolution (EvoX); CORAL multi-agent co-evolution |
| [Knowledge Accumulation](concepts/knowledge-accumulation.md) | Persistent memory across iterations: learnings.md, Cognition Base, Attempts/Notes/Skills; what makes loops compound |

---

## Key Entities (inline — no separate pages yet)

| Entity | Description | Referenced in |
|--------|-------------|---------------|
| Tau3 | Benchmark used by NeoSigma; baseline 0.560, achieved 0.780 | [auto-harness](sources/auto-harness.md) |
| TextArena | 145-game environment used by AutoHarness paper | [autoharness-arxiv](sources/autoharness-arxiv.md) |
| ARC-AGI | Reasoning benchmark; optimize_anything evolved 10→300 line agent on it | [optimize-anything](sources/optimize-anything.md) |
| NeoSigma AI | Org behind auto-harness repo and self-improving blog post | [auto-harness](sources/auto-harness.md) |
| Weco AI | Org behind AutoResearch | [autoresearch-vs-hpo](sources/autoresearch-vs-hpo.md) |
| GEPA | Genetic-Pareto + LLM framework from UC Berkeley/Stanford | [optimize-anything](sources/optimize-anything.md) |

---

## Source File Map

| Raw file in `sources/` | Wiki page |
|------------------------|-----------|
| `agent0` | [sources/agent0.md](sources/agent0.md) |
| `auto-harness` | [sources/auto-harness.md](sources/auto-harness.md) (repo) + [sources/autoharness-arxiv.md](sources/autoharness-arxiv.md) (paper) |
| `autoresearch-vs-hyperparams` | [sources/autoresearch-vs-hpo.md](sources/autoresearch-vs-hpo.md) |
| `meta-harness` | [sources/meta-harness.md](sources/meta-harness.md) |
| `optimize-anything` | [sources/optimize-anything.md](sources/optimize-anything.md) |
| `self-improving` | [sources/auto-harness.md](sources/auto-harness.md) (combined with repo) |
| `skydicover` | [sources/evox.md](sources/evox.md) |
| `autoagent` | [sources/autoagent-hkuds.md](sources/autoagent-hkuds.md) |
| `autoagent2` | [sources/autoagent-kevinrgu.md](sources/autoagent-kevinrgu.md) |
| `asi-evolve` | [sources/asi-evolve.md](sources/asi-evolve.md) |
| `coral` | [sources/coral.md](sources/coral.md) |
| `deep-research` | [sources/deep-research.md](sources/deep-research.md) |
| `agentflow` | [sources/agentflow.md](sources/agentflow.md) |
| `group-evolve` | [sources/group-evolve.md](sources/group-evolve.md) |
| `skill0` | [sources/skill-rl-skill0.md](sources/skill-rl-skill0.md) |
| `autogenesis` | [sources/autogenesis.md](sources/autogenesis.md) |
| `trace` | [sources/trace.md](sources/trace.md) |
| `webxskill` | [sources/webxskill.md](sources/webxskill.md) |
| `evoforge` | [sources/evoforge.md](sources/evoforge.md) |
| `honedhaiku` | [sources/honedhaiku.md](sources/honedhaiku.md) |
| `autoreason` | [sources/autoreason.md](sources/autoreason.md) |
| `halo` | [sources/halo.md](sources/halo.md) |
| `skillOpt` | [sources/skillopt.md](sources/skillopt.md) |
| `rlm_gepa` | [sources/rlm-gepa.md](sources/rlm-gepa.md) |
| `evo-hq` | [sources/evo.md](sources/evo.md) |

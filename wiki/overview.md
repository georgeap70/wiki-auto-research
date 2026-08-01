---
title: Self-Improving Agentic Systems — Overview
type: overview
tags: [self-improvement, agentic-ai, meta-learning, optimization]
sources: [agent0, auto-harness, autoresearch-vs-hpo, meta-harness, optimize-anything, optimize-anything-omni, neosigma-blog, evox, autoagent, autoagent2, asi-evolve, coral, deep-research, agentflow, group-evolve, skill0, autogenesis, trace, webxskill, evoforge, honedhaiku, autoreason, halo, skillopt, rlm-gepa, evo-hq, alphaevolve, shinkaevolve, squeeze-evolve, ophis, self-evolving]
last_updated: 2026-07-31
---

# Self-Improving Agentic Systems — Overview

A synthesis of current research and practice on agentic AI systems that improve themselves without human intervention.

## The Core Loop

Across all sources, the same fundamental cycle appears at different levels of abstraction:

```
measure → fail → propose → gate → repeat
```

1. **Measure**: Run the agent on a benchmark or production workload
2. **Fail**: Identify failures, cluster by root cause, extract diagnostic signal
3. **Propose**: Generate a candidate improvement (to prompts, code, architecture, or the optimization algorithm itself)
4. **Gate**: Accept the change only if it passes regression thresholds on held-out evals
5. **Repeat**: Iterate, compounding gains over time

## What Can Be Optimized

The key insight across this literature is that the optimization target can be anything expressible as text with a measurable outcome:

| Level | What changes | System |
|-------|-------------|--------|
| Task solutions | The agent's output on a specific task | [Agent0](sources/agent0.md) |
| Code policies / harnesses | Constraint-enforcement wrapper code | [AutoHarness (paper)](sources/autoharness-arxiv.md), [auto-harness (tool)](sources/auto-harness.md) |
| System prompts & scaffolding | The agent's harness and interaction patterns | [Meta-Harness](sources/meta-harness.md), [AutoAgent (KevinRGU)](sources/autoagent-kevinrgu.md), [Deep Research](sources/deep-research.md) |
| Agent harness via NL dialogue | Harness constructed/iterated via natural language | [AutoAgent (HKUDS)](sources/autoagent-hkuds.md) |
| Hyperparameters → architecture | Training configs, then structural code changes | [AutoResearch](sources/autoresearch-vs-hpo.md), [ASI-Evolve](sources/asi-evolve.md) |
| Data curation pipelines | Preprocessing strategies for training corpora | [ASI-Evolve](sources/asi-evolve.md) |
| RL algorithms | Advantage allocation, gradient computation | [ASI-Evolve](sources/asi-evolve.md) |
| Training interventions (mechanistic) | Tricks/edits derived from ~6,000 training-dynamics observables — no LLM, no search | [OPHIS](sources/ophis.md) |
| Whole codebases | Multi-file programs evolved by an LLM ensemble under evaluators | [AlphaEvolve](sources/alphaevolve.md), [ShinkaEvolve](sources/shinkaevolve.md) |
| Open-ended code solutions | Any code maximizing a grader score | [CORAL](sources/coral.md) |
| Agent policy weights (in-the-flow) | Planner weights updated during live execution | [AgentFlow](sources/agentflow.md) |
| Per-capability LoRA adapters | One adapter per identified capability gap | [TRACE](sources/trace.md) |
| Executable skills (parameterized programs) | Dual-mode skill artifacts with NL + code | [WebXSkill](sources/webxskill.md) |
| Typed agent resources (prompts/tools/memory) | Versioned resource modifications via protocol | [Autogenesis](sources/autogenesis.md) |
| Population of agent harnesses | Each agent in a parallel population hill-climbs its own `agent.py` | [EvoForge](sources/evoforge.md) |
| System prompt (only) | Prompt evolved by GEPA against PR-test-suite feedback | [HonedHaiku](sources/honedhaiku.md) |
| The output itself (per-query) | Inference-time tournament between incumbent / revision / synthesis | [AutoReason](sources/autoreason.md) |
| The output itself (per-query), cost-routed | Evolutionary refinement of answers, each step routed to a cheap/expensive model by per-instance difficulty | [Squeeze-Evolve](sources/squeeze-evolve.md) |
| Harness driven by production traces | OpenTelemetry traces → specialized RLM → coding-agent edits | [HALO](sources/halo.md) |
| Skill document as trainable state | Bounded add/delete/replace edits ("textual learning rate") to a markdown skill | [SkillOpt](sources/skillopt.md) |
| Skill instructions on an RLM runtime | GEPA proposes surgical edits to prose layered on top of a fixed RLM/DSPy structure; `AgentSpec` declares what's in-scope | [RLM-GEPA](sources/rlm-gepa.md) |
| Arbitrary repo metric via auto-discovery | `discover` skill instruments the benchmark; parallel subagents hill-climb under tree-search frontier strategies; gates inherit down the tree | [Evo](sources/evo.md) |
| The optimization algorithm itself | Which search strategy the optimizer uses | [EvoX](sources/evox.md) |
| The portfolio of optimizers | Which whole optimizer family / schedule to run | [optimize_anything Omni](sources/optimize-anything-omni.md) |

This progression — from task outputs → code policies → scaffolding → architecture → training data → learning algorithm → the optimizer → the portfolio of optimizers — represents increasing levels of meta-cognition in self-improvement. [ASI-Evolve](sources/asi-evolve.md) is the first system to target multiple levels (architecture + data + RL algorithm) simultaneously in a single automated loop; [optimize_anything Omni](sources/optimize-anything-omni.md) sits at the top, treating the *choice of optimizer itself* as composable — and finds empirically that no single optimizer dominates, so a portfolio-then-continue schedule beats any one engine.

An orthogonal way to read this whole table comes from [Xinming Tu's What×When taxonomy](sources/self-evolving.md): every row can also be indexed by *substrate* (external files / harness / weights) and *persistence horizon* (single session / across sessions / across users). The table above is essentially the "what changes" axis; Tu adds "how long it lasts," and a **consolidation path** — discoveries migrate files → harness → weights as they prove durable.

## Feedback Signals: Scalar vs. Rich

A recurring theme is that **rich diagnostic feedback substantially outperforms scalar reward signals**:

- [Meta-Harness](sources/meta-harness.md) provides the optimizer with full source code + execution traces (up to 10M tokens), enabling targeted diagnosis rather than black-box search
- [optimize_anything](sources/optimize-anything.md) formalizes this as **Actionable Side Information (ASI)** — compiler errors, profiler traces, test output — elevated to a first-class API concept
- [auto-harness (tool)](sources/auto-harness.md) stores persistent learnings across runs in `learnings.md`, allowing context recovery between optimization sessions
- [AutoHarness (paper)](sources/autoharness-arxiv.md) uses environmental feedback (legal/illegal move signals) to iteratively refine constraint code
- [HALO](sources/halo.md) makes the richest signal in the wiki — full OpenTelemetry production traces — tractable by inserting a *specialized trace-analysis RLM* between the raw traces and the harness-editor. The RLM's only job is to compress many traces into a diagnostic report; the coding agent then acts on the compressed signal
- [RLM-GEPA](sources/rlm-gepa.md) codifies the feedback contract: *"optimization quality is bounded by the evidence your metric returns."* Effective feedback must *name specific failures* (missing findings, unsupported claims, wrong cells) rather than *prescribe rewrites*. This is a clean restatement of the ASI thesis applied to skill-instruction optimization
- [Evo](sources/evo.md) runs RLM-inspired *cross-cutting scan subagents* between rounds: they read trace batches in parallel and surface compound failure patterns — explicitly *gate-failure intersections* and *shared root causes across traces*. Where HALO compresses production traffic and ASI-Evolve compresses experimental output, Evo compresses *within-loop* trace batches as a standing between-round phase
- [OPHIS](sources/ophis.md) pushes the signal a layer deeper still — past execution traces to **internal training dynamics**: ~6,000 tensor-level observables (norms, entropy-like measures, activation statistics) read *while the model trains*. Traces say what the program did; these say what the weights are doing. Strikingly, restricting an LLM to the *same* observable subspace nearly matched OPHIS on grokking — suggesting the *observable design* carries much of the value, independent of whether an LLM or a mechanistic reasoner consumes it
- [Squeeze-Evolve](sources/squeeze-evolve.md) uses feedback for a different job entirely: a **verifier-free difficulty proxy** (token-log-prob confidence or answer diversity) that estimates *how hard a problem is* so compute can be routed to a cheap or expensive model — signal-as-router rather than signal-as-critic

The implication: systems that explain *why* they failed improve faster than systems that only signal *how much* they failed. A corollary is becoming clear: when feedback is *too* rich, a dedicated compressor (a la HALO's RLM, or [ASI-Evolve](sources/asi-evolve.md)'s Analyzer) is itself a load-bearing component.

## Loop Architectures

### Single-agent self-edit
The agent edits its own code or prompts directly. Used by [auto-harness (tool)](sources/auto-harness.md) and [Meta-Harness](sources/meta-harness.md). Simpler, but the agent must reason about its own behavior.

### Two-agent co-evolution
A curriculum agent generates tasks; an executor agent solves them. Each improves the other. Used by [Agent0](sources/agent0.md). Eliminates need for human-curated training data — capability emerges from zero.

### Population-based / evolutionary
A population of candidates is maintained and evolved. Selection pressure applied via fitness functions. Used by [EvoX](sources/evox.md), [optimize_anything (GEPA)](sources/optimize-anything.md), and [EvoForge](sources/evoforge.md) (which adds a population dimension on top of the [AutoAgent (KevinRGU)](sources/autoagent-kevinrgu.md) `program.md → agent.py` hill-climb). Particularly powerful for discrete, structured search spaces.

### Tree-structured parallel hill-climb ([Evo](sources/evo.md))
A variant between single-thread hill-climb and flat population evolution. The orchestrator maintains a *tree* of committed experiments and, after each round, applies a *frontier strategy* (`argmax`, `top_k`, `epsilon_greedy`, `softmax`, `pareto_per_task`) to pick which committed branch to extend. Within a round, parallel subagents in isolated worktrees each form a hypothesis from shared state (failure traces, annotations, *discarded hypotheses*), edit, and benchmark. Between rounds, RLM-inspired scan subagents read trace batches in parallel and write cross-cutting findings back into shared state. The `pareto_per_task` strategy is credited to GEPA. This is the first wiki entry that packages multi-backend execution (worktree/pool/ssh/modal/e2b/daytona/aws/azure) and a dashboard as part of the loop primitive.

### Specialist optimizer / target separation
A recent architectural pattern: the *thing being improved* and the *thing doing the improving* are different specialized components. [HALO](sources/halo.md) separates an RLM trace-analyzer from a coding agent; [AutoReason](sources/autoreason.md) separates the author from independent blind judges; [SkillOpt](sources/skillopt.md) separates a frozen target model from a reflection-and-edit optimizer with its own meta-skill; [RLM-GEPA](sources/rlm-gepa.md) separates an *executor* (runs the RLM on examples, collects traces) from a *proposer* (reads scored traces, edits skill instructions). The motivation is similar across all four: the proposal agent and the evaluation/diagnosis agent have different jobs that fight each other when collapsed into one loop.

### Inference-time per-query tournaments ([AutoReason](sources/autoreason.md))
Not all self-improvement happens at training or deployment time. AutoReason runs a *per-query* refinement loop: each iteration generates an incumbent, an adversarial revision, and a synthesis; a fresh blind judge panel votes via Borda count; the loop terminates when the incumbent wins twice in a row. Tournament gating fixes three structural failures of vanilla critique-revise loops (prompt bias, scope creep, lack of restraint). This adds a third time axis — alongside per-deployment harness loops and long-horizon research loops — at which "improvement" can happen.

### Multi-agent co-evolution with shared memory ([CORAL](sources/coral.md))
Multiple autonomous agents explore in parallel, each running a full self-improvement loop, sharing a persistent three-layer knowledge store (Attempts/Notes/Skills). No external algorithm prescribes which candidates to retrieve — agents direct their own search. Emergent coordination behaviors arise from shared memory access alone: copycatting (agents adopt successful peer techniques), cross-referencing (synthesis of patterns across agents), and consensus formation (agents co-author notes at convergence). Four-agent runs outperform best-of-4 independent single-agent runs — gains are collaborative, not additive.

### Learn–Design–Experiment–Analyze ([ASI-Evolve](sources/asi-evolve.md))
A four-stage loop designed for long-horizon AI research tasks. The Analyzer agent compresses multi-dimensional experimental output into compact, decision-oriented reports before storage — solving the feedback complexity (D_feedback) problem that prevents naive loops from operating in GPU-hour research domains. A Cognition Base (embedding-indexed literature + prior discoveries) seeds each Design iteration with relevant prior knowledge.

### Meta-evolution (EvoX)
The evolution strategy itself is a candidate subject to evolution. An outer loop updates *how* candidates are generated; an inner loop generates candidates using the current strategy. The system can escape local optima by changing its own search operators.

### Portfolio-of-optimizers meta-loop ([optimize_anything Omni](sources/optimize-anything-omni.md))
One level above meta-evolution: rather than adapting the search strategy *inside* one optimizer, omni treats whole optimizer families ([GEPA](sources/optimize-anything.md), [AutoResearch](sources/autoresearch-vs-hpo.md), [Meta-Harness](sources/meta-harness.md)) as interchangeable engines behind one contract, races them in parallel on a shared budget, and continues the winner with a *fresh* optimizer to break plateaus. The empirical finding — no single optimizer dominates, but every portfolio beats every standalone — is EvoX's "no single strategy dominates" lifted to the optimizer level, and it is the direct counterweight to committing to one optimizer (see [experiment.md](experiment.md)).

### Mechanistic, non-search loop ([OPHIS](sources/ophis.md))
The one loop here that uses neither an LLM proposer nor a population. Its cycle — **Observation → Problem → Hypothesis → Intervention → Speed-up** — *deduces* interventions from a causal model of training dynamics instead of sampling and selecting. It generates candidates near-instantly and fails far less often than an LLM baseline (13.7% vs 42.1% on grokking), at the cost of being architecture-fixed (a "Training Copilot," not an architecture searcher). It is the wiki's sharpest statement that *understanding why* can substitute for *searching over what*. (OPHIS also carries its own causal-depth Stage 1/2/3 ladder — not to be confused with [CORAL's autonomy ladder](sources/coral.md).)

## Gating and Safety

Without regression gating, self-improvement risks catastrophic forgetting or proxy-metric overfitting:

- [auto-harness (tool)](sources/auto-harness.md) and **NeoSigma AI** use an **80% regression threshold**: changes that degrade previously passing tasks are rejected
- [AutoResearch](sources/autoresearch-vs-hpo.md) noted that classical HPO overfits to proxy metrics; the agentic approach avoids this by operating over a broader search space and using domain knowledge as implicit regularization
- [EvoX](sources/evox.md) uses stagnation detection to trigger strategy switches, avoiding premature convergence
- [optimize_anything](sources/optimize-anything.md) uses **Pareto-efficient multi-metric search** to avoid collapsing multiple objectives into a single scalar
- [Autogenesis](sources/autogenesis.md) proposes **auditable lineage + rollback** as protocol primitives — every entity (prompt, tool, memory) is versioned, every modification carries rationale, and any degradation can be reverted. Safety is built into the substrate rather than the optimizer
- [SkillOpt](sources/skillopt.md) uses a held-out validation gate *and* mines rejected edits as a negative-example buffer for the optimizer — failed proposals become structured negative signal rather than discarded noise (analogous to hard-negative mining)
- [AutoReason](sources/autoreason.md)'s Borda-count tournament is itself the gate: a change only lands when an independent judge panel ranks it above the incumbent, eliminating the "always revise" bias of vanilla self-refinement
- [Evo](sources/evo.md) treats gates as first-class primitives that **inherit down the experiment tree**: a gate at the root runs on every descendant; narrower gates attach to specific branches. Gate failure overrides score improvement (*"An experiment that fails a gate is discarded even if its score beats the current best"*), a stronger commitment than soft-threshold gating. The held-out-slice score-floor gate is *auto-attached* during the `discover` bootstrap, so even a naive user gets generalization protection by default

## Empirical Results

| System | Benchmark | Baseline | After | Gain |
|--------|-----------|---------|-------|------|
| auto-harness / NeoSigma | Tau3 | 0.560 | 0.780 | +39% |
| EvoX | 172 competitive programming problems | — | +34% median | — |
| AutoHarness | TextArena (145 games) | 78% loss rate from illegal moves | Smaller model beats larger without harness | — |
| Agent0 | Multiple reasoning benchmarks | Zero data start | Continuous improvement | — |
| AutoAgent (HKUDS) | GAIA (deep research) | — | Comparable to Claude 3.5 Sonnet | — |
| AutoAgent (KevinRGU) | — | — | No published numbers | — |
| ASI-Evolve | Neural architecture (1.3B/100B) | DeltaNet 51.04% | 52.01% | +0.97 (~3× best human improvement) |
| ASI-Evolve | Pretraining data curation | Nemotron-CC 40.17% | 44.13% | +3.96 avg; MMLU +18.64 |
| ASI-Evolve | RL algorithm (14B model, AIME24) | GRPO 20.00 | 31.67 | +11.67 |
| CORAL (single agent) | Erdős overlap | OpenEvolve baseline | 0.38089 | 2.5× faster; 7× fewer evals |
| CORAL (4-agent) | Kernel Engineering (cycles) | 1,363 | **1,103** | −18.3% cycles |
| CORAL (4-agent + search) | Polyominoes Packing | 87% (prev SOTA) | **89.4** | New SOTA |
| CORAL | OpenVaccine (ML engineering) | Top human score | +20.5% | 2 min, 2 evals |
| Deep Research (GEPA custom) | ScholarQA CS (minimal start) | 0.513 | **0.705** | +0.192 (beats expert-designed prompts) |
| TRACE | τ²-Bench | base agent | +14.1 pts; 47.0% vs GRPO 37.8% at 5,120 rollouts | +7.4 over strongest baseline |
| TRACE | ToolSandBox | base +4 perfect | **+7 perfect scores** | — |
| WebXSkill | WebArena | baseline | +9.8 pts | — |
| WebXSkill | WebVoyager | baseline | +12.9 pts | — |
| EvoForge | GPT-5-nano harness | baseline | 10× baseline, 2× Codex CLI | — |
| HonedHaiku | Bug-fixing holdout (Haiku 3.5) | 64.96% | **84.62%** | +19.66pp |
| AutoReason | CodeContests (Sonnet 4.6, 150 problems) | 73% | 77% | +4pp |
| AutoReason | vs best-of-6 sampling (matched compute) | 31% | **40%** | same budget, better outcome |
| HALO | AppWorld dev (Gemini 3 Flash) | 36.8% | 52.6% | +15.8pp (test +10.7pp) |
| HALO | AppWorld dev (Sonnet 4.6) | 73.7% | **89.5%** | +15.8pp (test +10.7pp) |
| SkillOpt | 7 models × 6 benchmarks | — | **best-or-tied-best 52/52** | avg 9–25% |
| SkillOpt | ALFWorld | 70.9% | 85.8% | +14.9pp |
| SkillOpt | cross-model skill transfer | — | +15.2% | strongest transfer evidence in the wiki |
| SkillOpt | cross-harness skill transfer | — | +31.8% | — |
| AlphaEvolve | Borg data-center scheduler (production) | prior heuristic | recovers 0.7% of global compute | in prod >1yr |
| AlphaEvolve | 4×4 complex matmul | Strassen (1969) | **48 scalar mults** | first improvement in 56 yrs |
| ShinkaEvolve | Circle packing | prior SOTA | new SOTA | **in 150 samples** |
| optimize_anything Omni | Frontier-CS (10 problems, $20 each) | GEPA 43.8 | **omni-GEPA 61.8 / omni-AutoResearch 63.2** | +41% on the weakest engine; every omni > every standalone |
| OPHIS | Grokking (modular addition) | LLM 57.9% substantial-improve, 42.1% fail | **72.9% substantial-improve, 13.7% fail** | 350 tricks tested |
| OPHIS | NanoGPT (already RSI-optimized) | Karpathy-autoresearch +0.001 (noise) | val BPB 0.93410 → 0.93184 | **−7.43σ** |

## Modular Decomposition of the Improvement Problem

A pattern emerging in the newest sources: **decompose the global improvement problem into narrower sub-problems with cleaner training signals**, then compose the results at inference. This contrasts with the end-to-end approach of optimizing one monolithic agent on one global reward.

- [TRACE](sources/trace.md): each capability gap becomes its own synthetic training environment with dense capability-isolated reward; one LoRA adapter per capability; a router composes them at inference
- [WebXSkill](sources/webxskill.md): each recurring web interaction pattern becomes a parameterized skill with dual grounded/guided deployment; URL-graph retrieval composes them per page context
- [SKILL-RL](sources/skill-rl-skill0.md): hierarchical SkillBank (general → task-specific) is composed per task; skills co-evolve with policy
- [CORAL](sources/coral.md)'s Skills store: NL + executable artifacts accumulated across agents; composition via agent-directed read

The common idea: when a global reward is sparse, *decompose into locally-dense sub-rewards*. The mechanism of decomposition (capabilities, skills, adapters) and the form of storage (weights, text, embeddings) differ, but the insight is consistent. This is an alternative to both [rich feedback](concepts/feedback-signals.md) (trace-based) and [credit assignment](sources/agentflow.md) (broadcast-based) approaches.

## Knowledge Accumulation as a First-Class Mechanism

A theme that emerges strongly from the newest sources: **persistent, structured knowledge accumulation** is what separates genuinely compounding self-improvement from random search. Approaches range from a flat file to typed multi-layer stores:

- [auto-harness](sources/auto-harness.md): flat `learnings.md` file, injected into context each session
- [ASI-Evolve](sources/asi-evolve.md): embedding-indexed Cognition Base (human priors + agent analyses); retrieved via semantic search
- [CORAL](sources/coral.md): three-layer store (Attempts for lineage, Notes for analysis, Skills for reusable procedures); shared across parallel agents
- [SkillOpt](sources/skillopt.md): single `best_skill.md` *is* the accumulated knowledge, plus a secondary store of rejected-edit negatives that informs future proposals
- [RLM-GEPA](sources/rlm-gepa.md): optimized skill instructions layered on top of a fixed RLM/DSPy structure; transfer across use cases is the explicit design goal, with `AgentSpec` declaring the transfer boundary

The right form of accumulation depends on the time horizon and the number of agents. Single-agent sequential loops benefit from simple document stores; multi-agent parallel runs require concurrent access and explicit distillation into transferable skills.

SkillOpt's transfer results (+15.2% cross-model, +31.8% cross-harness, +10.4% when used as the optimizer's own meta-skill) are the strongest evidence in the wiki that accumulated knowledge artifacts are not model- or harness-specific — i.e., that the "knowledge" being accumulated really is about the *task*, not about an incidental detail of how it was learned.

See [Knowledge Accumulation](concepts/knowledge-accumulation.md).

## The Productive Band (Goldilocks Zone) for Prompt Optimization

Two independent sources converged on the same shape: text-only optimization (no weight changes) has a baseline-dependent productive range.

| Baseline regime | What [HonedHaiku](sources/honedhaiku.md) saw | What [AutoReason](sources/autoreason.md) saw |
|-----------------|------------------------|------------------------|
| Very weak (<~50%) | Model can't execute complex methodologies; no gain | — |
| Productive (~50–70%) | +19.7pp on unseen bugs (Haiku 3.5: 65% → 85%) | Tournament gains largest here |
| Saturated (>~85%) | Prompt is no longer the bottleneck | Diminishing returns above ~60% on Haiku 4.5 |

[SkillOpt](sources/skillopt.md) partially contradicts this: it achieves best-or-tied-best across all 7 models including weaker ones. The likely reason is that its *bounded structured edits* are more learnable than free-form prompt mutations — i.e., constraining the edit space widens the productive band. This is consistent with SkillOpt's framing of edit-count as a *textual learning rate*: a smaller "step size" is what lets weaker models benefit.

## Open Questions

- How do self-improving systems avoid reward hacking at the meta-level (optimizing the optimizer)?
- What is the right granularity of human oversight — per-batch review, Pareto curve inspection, or fully autonomous?
- Can loop architectures compose? (e.g., Agent0-style co-evolution inside an EvoX-style meta-optimizer; CORAL agents running ASI-Evolve-style Analyze stages)
- How do rich diagnostic traces scale — 10M token context windows work now, but will this approach hit limits?
- ASI-Evolve demonstrates AI-discovered RL algorithms outperforming human-designed GRPO. If such algorithms are used to train the next generation of models, does that create a true recursive self-improvement loop?
- CORAL's emergent coordination (copycatting, consensus) emerges from shared memory alone — no communication protocol is hard-coded. What other coordination behaviors emerge at larger agent populations?
- [deep-research](sources/deep-research.md) shows that GEPA starting from minimal prompts can beat TextGrad starting from expert prompts. Does this generalize: are good optimization methods more valuable than good initializations?
- [TRACE](sources/trace.md) relies on supervising LLM agents to diagnose capability gaps and generate environments; can this diagnostic step itself be automated by the agent being improved, making the loop fully autonomous?
- [Autogenesis](sources/autogenesis.md) proposes a protocol where self-modification is a first-class primitive with lineage and rollback. Does such a protocol need industry adoption (like MCP) to matter, or can individual frameworks implement the ideas without a shared standard?
- Modular decomposition ([TRACE](sources/trace.md), skill libraries) vs. monolithic end-to-end training ([AgentFlow](sources/agentflow.md), SKILL-0): are these genuinely different tradeoffs, or does one strictly dominate as systems scale?
- The *optimizer/target separation* pattern ([HALO](sources/halo.md), [AutoReason](sources/autoreason.md), [SkillOpt](sources/skillopt.md), [RLM-GEPA](sources/rlm-gepa.md)) keeps reappearing. Is collapsing the proposal and the evaluation/diagnosis into one agent fundamentally limited, or just inconvenient at current model scales?
- [RLM-GEPA](sources/rlm-gepa.md)'s `AgentSpec` makes "what the optimizer needs to know that it can't infer" a typed, declared input. Should this become a first-class concept across the literature — every optimizer accompanied by a declared spec — or does it just push the prompt-engineering problem one level up?
- The MIT-CSAIL Recursive Language Model substrate underlies both [HALO](sources/halo.md) (RLM as trace compressor) and [RLM-GEPA](sources/rlm-gepa.md) (RLM as runtime). Will "use an RLM" become the default scaffold for production agents the way "use a transformer" became for models — and if so, do harness-optimization techniques designed for non-RLM agents transfer cleanly?
- [SkillOpt](sources/skillopt.md) treats edit count as a "textual learning rate". Are there analogues for other gradient-descent hyperparameters (momentum, weight decay, schedules) in the text-optimization regime?
- [HALO](sources/halo.md)'s OpenTelemetry-driven loop assumes you have production traffic to learn from. For agents that don't yet have users, what's the equivalent? Synthetic traffic from an adversarial agent? Curriculum from a companion?
- Inference-time loops like [AutoReason](sources/autoreason.md) sit beside training-time and deployment-time loops. Should these three time scales compose (per-query refinement *inside* per-deployment harness optimization *inside* long-horizon architecture search), or do their objectives interfere?
- [Evo](sources/evo.md) is one of the first packaged orchestrators for [Karpathy-style autoresearch](sources/autoresearch-vs-hpo.md). Does the *tree*-shaped exploration (with configurable frontier strategies) genuinely beat flat-population evolution ([EvoForge](sources/evoforge.md), [Group-Evolving Agents](sources/group-evolve.md)) in practice, or is the tree mostly a UX/lineage win that doesn't change the search outcomes?
- Evo's *discarded-hypothesis* bucket is unusual — most systems retain only successful branches. [SkillOpt](sources/skillopt.md) mined rejected text edits as a negative-signal buffer; Evo does this at the granularity of *experimental directions*. Does negative-hypothesis storage become a standard piece of population-based agentic search, the way replay buffers became standard in deep RL?
- [optimize_anything Omni](sources/optimize-anything-omni.md) finds that a *portfolio* of optimizers plus a fresh-optimizer restart beats any single optimizer on Frontier-CS. Is "portfolio-then-continue" a general free lunch, or an artifact of competitive-programming tasks with steep early plateaus? Does the gain survive when all engines share the same base model, or is engine *diversity* the real source of the lift? And how does it interact with [experiment.md](experiment.md)'s decision to commit to a single GEPA loop?
- [OPHIS](sources/ophis.md) argues that *understanding why* (mechanistic reasoning over training dynamics) can substitute for *searching over what* (LLM/evolutionary proposal) — with far lower failure rates but a fixed architecture. Can the two be combined: feed OPHIS-style internal-dynamics observables as [ASI](sources/optimize-anything.md) into an LLM or evolutionary proposer? Its own result (an LLM on the same observable subspace nearly matched it) suggests the observable design, not the reasoner, is the lever — does that generalize beyond grokking/NanoGPT?
- Cost-aware model routing now appears at three distinct layers: [ShinkaEvolve](sources/shinkaevolve.md)'s operator-level UCB bandit, [Squeeze-Evolve](sources/squeeze-evolve.md)'s per-instance difficulty routing, and [experiment.md](experiment.md)'s per-stage `[prompt, model]` co-optimization. Are these substitutes or complements — could a single system route cheap-vs-expensive at the operator, the instance, *and* the pipeline-stage level at once?
- [Xinming Tu's taxonomy](sources/self-evolving.md) makes plain that this wiki is dense in the *harness × across-sessions* cell and sparse in *across-users*. Is the population-level flywheel (discoveries that benefit every user) the field's next frontier, and does it necessarily require the *weights* substrate — or can shared files/skills ([CORAL](sources/coral.md)'s Skills, Agent Skills) carry most of it?

## See Also

- [Self-Improvement Loop](concepts/self-improvement-loop.md) — the core measure-fail-propose-gate cycle in detail
- [Feedback Signals](concepts/feedback-signals.md) — scalar vs. rich diagnostic feedback
- [Harness Optimization](concepts/harness-optimization.md) — optimizing the code wrapper around an agent
- [Evolutionary Optimization](concepts/evolutionary-optimization.md) — population-based and meta-evolutionary approaches
- [Regression Gating](concepts/regression-gating.md) — how safe self-improvement is enforced
- [Self-Evolving Agents (Tu)](sources/self-evolving.md) — a companion What×When taxonomy that indexes every entry above by substrate and persistence horizon

---
title: Self-Improving Agentic Systems — Overview
type: overview
tags: [self-improvement, agentic-ai, meta-learning, optimization]
sources: [agent0, auto-harness, autoresearch-vs-hpo, meta-harness, optimize-anything, neosigma-blog, evox, autoagent, autoagent2, asi-evolve, coral, deep-research, agentflow, group-evolve, skill0, autogenesis, trace, webxskill, evoforge, honedhaiku, autoreason, halo, skillopt]
last_updated: 2026-05-26
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
| Task solutions | The agent's output on a specific task | [[sources/agent0|Agent0]] |
| Code policies / harnesses | Constraint-enforcement wrapper code | [[sources/autoharness-arxiv|AutoHarness (paper)]], [[sources/auto-harness|auto-harness (tool)]] |
| System prompts & scaffolding | The agent's harness and interaction patterns | [[sources/meta-harness|Meta-Harness]], [[sources/autoagent-kevinrgu|AutoAgent (KevinRGU)]], [[sources/deep-research|Deep Research]] |
| Agent harness via NL dialogue | Harness constructed/iterated via natural language | [[sources/autoagent-hkuds|AutoAgent (HKUDS)]] |
| Hyperparameters → architecture | Training configs, then structural code changes | [[sources/autoresearch-vs-hpo|AutoResearch]], [[sources/asi-evolve|ASI-Evolve]] |
| Data curation pipelines | Preprocessing strategies for training corpora | [[sources/asi-evolve|ASI-Evolve]] |
| RL algorithms | Advantage allocation, gradient computation | [[sources/asi-evolve|ASI-Evolve]] |
| Open-ended code solutions | Any code maximizing a grader score | [[sources/coral|CORAL]] |
| Agent policy weights (in-the-flow) | Planner weights updated during live execution | [[sources/agentflow|AgentFlow]] |
| Per-capability LoRA adapters | One adapter per identified capability gap | [[sources/trace|TRACE]] |
| Executable skills (parameterized programs) | Dual-mode skill artifacts with NL + code | [[sources/webxskill|WebXSkill]] |
| Typed agent resources (prompts/tools/memory) | Versioned resource modifications via protocol | [[sources/autogenesis|Autogenesis]] |
| Population of agent harnesses | Each agent in a parallel population hill-climbs its own `agent.py` | [[sources/evoforge|EvoForge]] |
| System prompt (only) | Prompt evolved by GEPA against PR-test-suite feedback | [[sources/honedhaiku|HonedHaiku]] |
| The output itself (per-query) | Inference-time tournament between incumbent / revision / synthesis | [[sources/autoreason|AutoReason]] |
| Harness driven by production traces | OpenTelemetry traces → specialized RLM → coding-agent edits | [[sources/halo|HALO]] |
| Skill document as trainable state | Bounded add/delete/replace edits ("textual learning rate") to a markdown skill | [[sources/skillopt|SkillOpt]] |
| The optimization algorithm itself | Which search strategy the optimizer uses | [[sources/evox|EvoX]] |

This progression — from task outputs → code policies → scaffolding → architecture → training data → learning algorithm → the optimizer — represents increasing levels of meta-cognition in self-improvement. [[sources/asi-evolve|ASI-Evolve]] is the first system to target multiple levels (architecture + data + RL algorithm) simultaneously in a single automated loop.

## Feedback Signals: Scalar vs. Rich

A recurring theme is that **rich diagnostic feedback substantially outperforms scalar reward signals**:

- [[sources/meta-harness|Meta-Harness]] provides the optimizer with full source code + execution traces (up to 10M tokens), enabling targeted diagnosis rather than black-box search
- [[sources/optimize-anything|optimize_anything]] formalizes this as **Actionable Side Information (ASI)** — compiler errors, profiler traces, test output — elevated to a first-class API concept
- [[sources/auto-harness|auto-harness (tool)]] stores persistent learnings across runs in `learnings.md`, allowing context recovery between optimization sessions
- [[sources/autoharness-arxiv|AutoHarness (paper)]] uses environmental feedback (legal/illegal move signals) to iteratively refine constraint code
- [[sources/halo|HALO]] makes the richest signal in the wiki — full OpenTelemetry production traces — tractable by inserting a *specialized trace-analysis RLM* between the raw traces and the harness-editor. The RLM's only job is to compress many traces into a diagnostic report; the coding agent then acts on the compressed signal

The implication: systems that explain *why* they failed improve faster than systems that only signal *how much* they failed. A corollary is becoming clear: when feedback is *too* rich, a dedicated compressor (a la HALO's RLM, or [[sources/asi-evolve|ASI-Evolve]]'s Analyzer) is itself a load-bearing component.

## Loop Architectures

### Single-agent self-edit
The agent edits its own code or prompts directly. Used by [[sources/auto-harness|auto-harness (tool)]] and [[sources/meta-harness|Meta-Harness]]. Simpler, but the agent must reason about its own behavior.

### Two-agent co-evolution
A curriculum agent generates tasks; an executor agent solves them. Each improves the other. Used by [[sources/agent0|Agent0]]. Eliminates need for human-curated training data — capability emerges from zero.

### Population-based / evolutionary
A population of candidates is maintained and evolved. Selection pressure applied via fitness functions. Used by [[sources/evox|EvoX]], [[sources/optimize-anything|optimize_anything (GEPA)]], and [[sources/evoforge|EvoForge]] (which adds a population dimension on top of the [[sources/autoagent-kevinrgu|AutoAgent (KevinRGU)]] `program.md → agent.py` hill-climb). Particularly powerful for discrete, structured search spaces.

### Specialist optimizer / target separation
A recent architectural pattern: the *thing being improved* and the *thing doing the improving* are different specialized components. [[sources/halo|HALO]] separates an RLM trace-analyzer from a coding agent; [[sources/autoreason|AutoReason]] separates the author from independent blind judges; [[sources/skillopt|SkillOpt]] separates a frozen target model from a reflection-and-edit optimizer with its own meta-skill. The motivation is similar across all three: the proposal agent and the evaluation/diagnosis agent have different jobs that fight each other when collapsed into one loop.

### Inference-time per-query tournaments ([[sources/autoreason|AutoReason]])
Not all self-improvement happens at training or deployment time. AutoReason runs a *per-query* refinement loop: each iteration generates an incumbent, an adversarial revision, and a synthesis; a fresh blind judge panel votes via Borda count; the loop terminates when the incumbent wins twice in a row. Tournament gating fixes three structural failures of vanilla critique-revise loops (prompt bias, scope creep, lack of restraint). This adds a third time axis — alongside per-deployment harness loops and long-horizon research loops — at which "improvement" can happen.

### Multi-agent co-evolution with shared memory ([[sources/coral|CORAL]])
Multiple autonomous agents explore in parallel, each running a full self-improvement loop, sharing a persistent three-layer knowledge store (Attempts/Notes/Skills). No external algorithm prescribes which candidates to retrieve — agents direct their own search. Emergent coordination behaviors arise from shared memory access alone: copycatting (agents adopt successful peer techniques), cross-referencing (synthesis of patterns across agents), and consensus formation (agents co-author notes at convergence). Four-agent runs outperform best-of-4 independent single-agent runs — gains are collaborative, not additive.

### Learn–Design–Experiment–Analyze ([[sources/asi-evolve|ASI-Evolve]])
A four-stage loop designed for long-horizon AI research tasks. The Analyzer agent compresses multi-dimensional experimental output into compact, decision-oriented reports before storage — solving the feedback complexity (D_feedback) problem that prevents naive loops from operating in GPU-hour research domains. A Cognition Base (embedding-indexed literature + prior discoveries) seeds each Design iteration with relevant prior knowledge.

### Meta-evolution (EvoX)
The evolution strategy itself is a candidate subject to evolution. An outer loop updates *how* candidates are generated; an inner loop generates candidates using the current strategy. The system can escape local optima by changing its own search operators.

## Gating and Safety

Without regression gating, self-improvement risks catastrophic forgetting or proxy-metric overfitting:

- [[sources/auto-harness|auto-harness (tool)]] and **NeoSigma AI** use an **80% regression threshold**: changes that degrade previously passing tasks are rejected
- [[sources/autoresearch-vs-hpo|AutoResearch]] noted that classical HPO overfits to proxy metrics; the agentic approach avoids this by operating over a broader search space and using domain knowledge as implicit regularization
- [[sources/evox|EvoX]] uses stagnation detection to trigger strategy switches, avoiding premature convergence
- [[sources/optimize-anything|optimize_anything]] uses **Pareto-efficient multi-metric search** to avoid collapsing multiple objectives into a single scalar
- [[sources/autogenesis|Autogenesis]] proposes **auditable lineage + rollback** as protocol primitives — every entity (prompt, tool, memory) is versioned, every modification carries rationale, and any degradation can be reverted. Safety is built into the substrate rather than the optimizer
- [[sources/skillopt|SkillOpt]] uses a held-out validation gate *and* mines rejected edits as a negative-example buffer for the optimizer — failed proposals become structured negative signal rather than discarded noise (analogous to hard-negative mining)
- [[sources/autoreason|AutoReason]]'s Borda-count tournament is itself the gate: a change only lands when an independent judge panel ranks it above the incumbent, eliminating the "always revise" bias of vanilla self-refinement

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

## Modular Decomposition of the Improvement Problem

A pattern emerging in the newest sources: **decompose the global improvement problem into narrower sub-problems with cleaner training signals**, then compose the results at inference. This contrasts with the end-to-end approach of optimizing one monolithic agent on one global reward.

- [[sources/trace|TRACE]]: each capability gap becomes its own synthetic training environment with dense capability-isolated reward; one LoRA adapter per capability; a router composes them at inference
- [[sources/webxskill|WebXSkill]]: each recurring web interaction pattern becomes a parameterized skill with dual grounded/guided deployment; URL-graph retrieval composes them per page context
- [[sources/skill-rl-skill0|SKILL-RL]]: hierarchical SkillBank (general → task-specific) is composed per task; skills co-evolve with policy
- [[sources/coral|CORAL]]'s Skills store: NL + executable artifacts accumulated across agents; composition via agent-directed read

The common idea: when a global reward is sparse, *decompose into locally-dense sub-rewards*. The mechanism of decomposition (capabilities, skills, adapters) and the form of storage (weights, text, embeddings) differ, but the insight is consistent. This is an alternative to both [[concepts/feedback-signals|rich feedback]] (trace-based) and [[sources/agentflow|credit assignment]] (broadcast-based) approaches.

## Knowledge Accumulation as a First-Class Mechanism

A theme that emerges strongly from the newest sources: **persistent, structured knowledge accumulation** is what separates genuinely compounding self-improvement from random search. Approaches range from a flat file to typed multi-layer stores:

- [[sources/auto-harness|auto-harness]]: flat `learnings.md` file, injected into context each session
- [[sources/asi-evolve|ASI-Evolve]]: embedding-indexed Cognition Base (human priors + agent analyses); retrieved via semantic search
- [[sources/coral|CORAL]]: three-layer store (Attempts for lineage, Notes for analysis, Skills for reusable procedures); shared across parallel agents
- [[sources/skillopt|SkillOpt]]: single `best_skill.md` *is* the accumulated knowledge, plus a secondary store of rejected-edit negatives that informs future proposals

The right form of accumulation depends on the time horizon and the number of agents. Single-agent sequential loops benefit from simple document stores; multi-agent parallel runs require concurrent access and explicit distillation into transferable skills.

SkillOpt's transfer results (+15.2% cross-model, +31.8% cross-harness, +10.4% when used as the optimizer's own meta-skill) are the strongest evidence in the wiki that accumulated knowledge artifacts are not model- or harness-specific — i.e., that the "knowledge" being accumulated really is about the *task*, not about an incidental detail of how it was learned.

See [[concepts/knowledge-accumulation]].

## The Productive Band (Goldilocks Zone) for Prompt Optimization

Two independent sources converged on the same shape: text-only optimization (no weight changes) has a baseline-dependent productive range.

| Baseline regime | What [[sources/honedhaiku|HonedHaiku]] saw | What [[sources/autoreason|AutoReason]] saw |
|-----------------|------------------------|------------------------|
| Very weak (<~50%) | Model can't execute complex methodologies; no gain | — |
| Productive (~50–70%) | +19.7pp on unseen bugs (Haiku 3.5: 65% → 85%) | Tournament gains largest here |
| Saturated (>~85%) | Prompt is no longer the bottleneck | Diminishing returns above ~60% on Haiku 4.5 |

[[sources/skillopt|SkillOpt]] partially contradicts this: it achieves best-or-tied-best across all 7 models including weaker ones. The likely reason is that its *bounded structured edits* are more learnable than free-form prompt mutations — i.e., constraining the edit space widens the productive band. This is consistent with SkillOpt's framing of edit-count as a *textual learning rate*: a smaller "step size" is what lets weaker models benefit.

## Open Questions

- How do self-improving systems avoid reward hacking at the meta-level (optimizing the optimizer)?
- What is the right granularity of human oversight — per-batch review, Pareto curve inspection, or fully autonomous?
- Can loop architectures compose? (e.g., Agent0-style co-evolution inside an EvoX-style meta-optimizer; CORAL agents running ASI-Evolve-style Analyze stages)
- How do rich diagnostic traces scale — 10M token context windows work now, but will this approach hit limits?
- ASI-Evolve demonstrates AI-discovered RL algorithms outperforming human-designed GRPO. If such algorithms are used to train the next generation of models, does that create a true recursive self-improvement loop?
- CORAL's emergent coordination (copycatting, consensus) emerges from shared memory alone — no communication protocol is hard-coded. What other coordination behaviors emerge at larger agent populations?
- [[sources/deep-research|deep-research]] shows that GEPA starting from minimal prompts can beat TextGrad starting from expert prompts. Does this generalize: are good optimization methods more valuable than good initializations?
- [[sources/trace|TRACE]] relies on supervising LLM agents to diagnose capability gaps and generate environments; can this diagnostic step itself be automated by the agent being improved, making the loop fully autonomous?
- [[sources/autogenesis|Autogenesis]] proposes a protocol where self-modification is a first-class primitive with lineage and rollback. Does such a protocol need industry adoption (like MCP) to matter, or can individual frameworks implement the ideas without a shared standard?
- Modular decomposition ([[sources/trace|TRACE]], skill libraries) vs. monolithic end-to-end training ([[sources/agentflow|AgentFlow]], SKILL-0): are these genuinely different tradeoffs, or does one strictly dominate as systems scale?
- The *optimizer/target separation* pattern ([[sources/halo|HALO]], [[sources/autoreason|AutoReason]], [[sources/skillopt|SkillOpt]]) keeps reappearing. Is collapsing the proposal and the evaluation/diagnosis into one agent fundamentally limited, or just inconvenient at current model scales?
- [[sources/skillopt|SkillOpt]] treats edit count as a "textual learning rate". Are there analogues for other gradient-descent hyperparameters (momentum, weight decay, schedules) in the text-optimization regime?
- [[sources/halo|HALO]]'s OpenTelemetry-driven loop assumes you have production traffic to learn from. For agents that don't yet have users, what's the equivalent? Synthetic traffic from an adversarial agent? Curriculum from a companion?
- Inference-time loops like [[sources/autoreason|AutoReason]] sit beside training-time and deployment-time loops. Should these three time scales compose (per-query refinement *inside* per-deployment harness optimization *inside* long-horizon architecture search), or do their objectives interfere?

## See Also

- [[concepts/self-improvement-loop]] — the core measure-fail-propose-gate cycle in detail
- [[concepts/feedback-signals]] — scalar vs. rich diagnostic feedback
- [[concepts/harness-optimization]] — optimizing the code wrapper around an agent
- [[concepts/evolutionary-optimization]] — population-based and meta-evolutionary approaches
- [[concepts/regression-gating]] — how safe self-improvement is enforced

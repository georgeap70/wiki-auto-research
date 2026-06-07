---
title: Harness Optimization
type: concept
tags: [harness, system-prompt, scaffolding, code-synthesis, constraint-enforcement]
sources: [auto-harness, meta-harness, autoharness-arxiv, autoagent, autoagent2, evoforge, honedhaiku, halo, skillOpt, rlm_gepa, evo-hq]
last_updated: 2026-06-06
---

# Harness Optimization

"Harness" refers to everything that wraps a base LLM: system prompts, tool definitions, interaction protocols, output parsers, constraint enforcement code, and any other scaffolding. Harness optimization is the practice of **automatically improving this wrapper** without modifying the underlying model weights.

## Why Harness Optimization Matters

- Model weights are expensive to change (retraining, fine-tuning)
- Harnesses are cheap to modify (text files, code files)
- Substantial capability gains are available at the harness layer
- A single harness update can improve performance across all tasks

[sources/auto-harness](../sources/auto-harness.md) demonstrated +39% gain on Tau3 by iterating the harness alone, with the underlying model (GPT-5.4) held constant. This suggests the harness is a high-leverage optimization target.

## Types of Harness Components

| Component | Example | Can be auto-optimized? |
|-----------|---------|----------------------|
| System prompt | Task description, instructions, persona | Yes |
| Tool definitions | Function signatures, descriptions | Yes |
| Interaction protocol | How to structure turns | Yes |
| Constraint enforcement code | Valid action checking | Yes ([sources/autoharness-arxiv](../sources/autoharness-arxiv.md)) |
| Output parser | Response parsing logic | Yes |
| Memory / learnings store | `learnings.md` | Emergent — the agent writes to it |

## Approaches in This Wiki

### auto-harness (NeoSigma AI) — [sources/auto-harness](../sources/auto-harness.md)
- Optimizes: system prompt + tool definitions
- Method: agent edits its own `agent/agent.py`, gates with regression suite
- Key feature: persistent `learnings.md` for cross-run memory
- Result: +39% on Tau3

### Meta-Harness (Stanford) — [sources/meta-harness](../sources/meta-harness.md)
- Optimizes: full harness (prompt + scaffolding)
- Method: optimizer agent reads full source + execution traces, proposes surgical edits
- Key feature: rich diagnostic context (10M tokens)
- Key claim: rich traces > scalar reward for proposal quality

### AutoHarness (Google DeepMind) — [sources/autoharness-arxiv](../sources/autoharness-arxiv.md)
- Optimizes: constraint enforcement code specifically
- Method: iterative synthesis from environmental action feedback
- Key insight: smaller models with good harnesses outperform larger models without
- Use case: constrained action environments (games, formal systems)

### AutoAgent (KevinRGU) — [sources/autoagent-kevinrgu](../sources/autoagent-kevinrgu.md)
- Optimizes: full harness (system prompt, tools, routing)
- Method: meta-agent hill-climbing loop; reads `program.md` directive, edits `agent.py`, scores on benchmark
- Key feature: explicit abstraction boundary — human writes intent in `program.md`, meta-agent owns implementation in `agent.py`
- Result: no published benchmark numbers

### AutoAgent (HKUDS) — [sources/autoagent-hkuds](../sources/autoagent-hkuds.md)
- Optimizes: full harness (agents, tools, workflows)
- Method: natural language dialogue — human specifies and iterates agent behavior conversationally, system auto-generates implementation
- Key feature: NL as the complete optimization interface; self-developing architecture with iterative feedback
- Result: GAIA performance comparable to Claude 3.5 Sonnet

### EvoForge (Haize Labs) — [sources/evoforge](../sources/evoforge.md)
- Optimizes: full harness (system prompt, tools, orchestration)
- Method: population of agents each running the `program.md → agent.py` hill-climbing loop in parallel; knowledge synthesis after each generation
- Key feature: three-tier abstraction (`evolve.md` → population; `program.md` → agent intent; `agent.py` → implementation); adds population dimension to [sources/autoagent-kevinrgu](../sources/autoagent-kevinrgu.md)
- Result: 2× Codex CLI, 10× baseline (GPT-5-nano)

### HonedHaiku (Tim Waldin) — [sources/honedhaiku](../sources/honedhaiku.md)
- Optimizes: system prompt only (prose-level)
- Method: GEPA mutation-selection loop; Agentelo scores against real PR test suites
- Key feature: "Goldilocks band" — prompt optimization only moves performance in the 50–70% range; training diversity critical for holdout generalization
- Result: +19.7pp on unseen bugs (65% → 85%); converged in 4 iterations

### HALO (Context Labs) — [sources/halo](../sources/halo.md)
- Optimizes: full harness (prompts, tool wiring, infrastructure)
- Method: production OpenTelemetry traces → specialized Recursive Language Model engine analyzes cross-trace patterns → coding agent edits harness → redeploy
- Key feature: explicit separation between **trace analysis** (RLM) and **harness editing** (coding agent); methodology designed around live deployment traffic
- Result: AppWorld +15.8pp dev for both Gemini 3 Flash and Sonnet 4.6; +10.7pp test (generalizes)

### SkillOpt (Microsoft Research) — [sources/skillopt](../sources/skillopt.md)
- Optimizes: a structured skill document (`best_skill.md`) treated as the model's "trainable state"
- Method: rollout → reflection (separate success/failure analysis) → bounded add/delete/replace edits (edit budget as "textual learning rate") → held-out validation gate
- Key feature: rejected-edit buffer as negative signal; optimizer-side meta-skill; strongest cross-model/cross-harness transfer evidence in the wiki
- Result: best-or-tied-best on 52/52 model×benchmark combinations; ALFWorld 70.9% → 85.8%; +15.2% cross-model, +31.8% cross-harness transfer without re-optimization

### Evo (evo-hq) — [sources/evo](../sources/evo.md)
- Optimizes: any code metric `discover` can identify in the repo (e.g., parser speed, accuracy on a held-out task)
- Method: `discover` instruments the benchmark; parallel subagents in isolated git worktrees hill-climb against it; tree-search over committed branches with configurable frontier strategy (`argmax`/`top_k`/`epsilon_greedy`/`softmax`/`pareto_per_task`); RLM-inspired cross-cutting scan subagents read trace batches between rounds and write findings to shared state
- Key feature: gates as first-class primitives that **inherit down the experiment tree**; gate failure overrides score improvement; held-out-slice score-floor gate auto-attached during `discover`; 8 execution backends (worktree/pool/ssh/modal/e2b/daytona/aws/azure); dashboard for inspecting the tree and tuning frontier strategy
- Result: no published benchmark numbers — packaged orchestrator rather than a research artifact

### RLM-GEPA (Trampoline AI) — [sources/rlm-gepa](../sources/rlm-gepa.md)
- Optimizes: skill instructions (prose) layered on top of a fixed Recursive Language Model runtime; DSPy signatures and tools are held constant
- Method: executor loop runs the RLM and collects `RunTrace` + score + feedback; proposer loop reads scored traces and issues GEPA-style surgical edits; an `AgentSpec` declares what behavioral changes are in-scope
- Key feature: **AgentSpec** as a typed, declared description of optimization context (use cases, runtime affordances, scoring methodology, counterfactual axes) — solves the "what should the optimizer know that it can't infer" problem; evidence-bounded feedback contract (name failures, don't prescribe rewrites)
- Result: no published benchmarks — positioned as production-grade infrastructure

## Comparison

| System | Feedback type | Scope | Human involvement |
|--------|---------------|-------|------------------|
| auto-harness | Production traces + pass/fail | Prompt + tools | None (overnight) |
| Meta-Harness | Full traces + source code | Full harness | None |
| AutoHarness | Environmental action signals | Constraint code only | None |
| AutoAgent (KevinRGU) | Benchmark score (scalar) | Full harness | Write `program.md` once |
| AutoAgent (HKUDS) | Conversational feedback | Full harness + workflows | NL direction per change |
| EvoForge | Benchmark score (scalar, population) | Full harness | Write `evolve.md` + `program.md` |
| HonedHaiku | PR test suite pass/fail | System prompt | Define challenges once |
| HALO | OpenTelemetry production traces | Full harness | Deploy + define benchmarks |
| SkillOpt | Rollout scores + rejected-edit buffer | Structured skill document | Define benchmarks once |
| RLM-GEPA | RunTrace + score + failure-description feedback | Skill instructions over a fixed RLM/DSPy structure | Author RLM, dataset, metric, and AgentSpec |
| Evo | Score + gate pass/fail + cross-cutting scan findings (gate intersections, shared root causes) + shared discarded-hypothesis bucket | Any repo metric, via auto-discovered benchmark | Run `/evo:discover` once; configure frontier strategy; optionally pause between rounds |

## Connections

- [concepts/self-improvement-loop](self-improvement-loop.md) — harness optimization is the loop applied at the scaffolding layer
- [concepts/feedback-signals](feedback-signals.md) — rich traces vs. scalar pass/fail determines proposal quality
- [concepts/regression-gating](regression-gating.md) — all three systems use gating to prevent regressions
- [sources/optimize-anything](../sources/optimize-anything.md) — generalizes harness optimization to any text artifact

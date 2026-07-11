# Self-Improving Agentic Systems Wiki — Schema

This wiki tracks research, tools, and ideas around the self-improvement of agentic AI systems.

## Directory Structure

```
self-improvement/
  CLAUDE.md               ← this file (schema + instructions)
  sources/                ← raw source files (immutable — never modify)
  wiki/
    index.md              ← catalog of all wiki pages
    log.md                ← append-only ingest/query/lint log
    overview.md           ← high-level synthesis of the field
    sources/              ← one summary page per raw source
    concepts/             ← concept and mechanism pages
    entities/             ← pages for specific systems, papers, orgs
```

## Conventions

### Frontmatter
All wiki pages should include YAML frontmatter:
```yaml
---
title: Page Title
type: source | concept | entity | overview
tags: [tag1, tag2]
sources: [source-slug1, source-slug2]   # which raw sources inform this page
last_updated: YYYY-MM-DD
---
```

### Page types
- **source**: Summary of a single raw source (paper, blog post, repo). One per raw source file.
- **concept**: A mechanism, technique, or idea that appears across multiple sources (e.g., "regression gating", "feedback signals").
- **entity**: A specific system, paper, organization, or benchmark (e.g., "Agent0", "NeoSigma AI", "Tau3").
- **overview**: High-level synthesis spanning the whole topic.

### Cross-references
Use standard Obsidian wiki links: `[[page-name]]`. Always link to concept and entity pages when first mentioned in a page.

### Source slugs
Raw source files → wiki source pages:
| Raw file                   | Wiki source page               |
|----------------------------|-------------------------------|
| agent0                     | wiki/sources/agent0.md        |
| auto-harness               | wiki/sources/auto-harness.md  |
| autoresearch-vs-hyperparams| wiki/sources/autoresearch-vs-hpo.md |
| meta-harness               | wiki/sources/meta-harness.md  |
| optimize-anything          | wiki/sources/optimize-anything.md |
| self-improving             | wiki/sources/neosigma-blog.md |
| skydicover                 | wiki/sources/evox.md          |
| autoagent                  | wiki/sources/autoagent-hkuds.md |
| autoagent2                 | wiki/sources/autoagent-kevinrgu.md |
| rlm_gepa                   | wiki/sources/rlm-gepa.md      |
| evo-hq                     | wiki/sources/evo.md           |
| self-harness               | wiki/sources/self-harness.md  |
| hf-harness                 | wiki/sources/evolve-the-harness.md |
| interaction-trajectory-mining | wiki/sources/interaction-trajectory-mining.md |
| weng-blog                  | wiki/sources/weng-harness-blog.md |
| stop                       | wiki/sources/stop.md          |
| adas                       | wiki/sources/adas.md          |
| aflow                      | wiki/sources/aflow.md         |
| dgm                        | wiki/sources/dgm.md           |
| ace                        | wiki/sources/ace.md           |
| mce                        | wiki/sources/mce.md           |
| hyperagents                | wiki/sources/hyperagents.md   |

## Workflows

### Ingest a new source
1. Drop the URL/file in `sources/`
2. Fetch and read the source
3. Create a summary page in `wiki/sources/`
4. Update `wiki/index.md`
5. Update relevant concept and entity pages (or create new ones)
6. Append an entry to `wiki/log.md`

### Query
1. Read `wiki/index.md` to find relevant pages
2. Read those pages and synthesize an answer
3. If the answer is non-trivial, save it as a new page or append to an existing one
4. Log the query in `wiki/log.md`

### Lint
Periodically check for: orphan pages, missing cross-references, stale claims superseded by newer sources, concepts mentioned but lacking their own page.

## Domain Notes

The core topic is: **how can agentic AI systems improve themselves without human intervention?**

Key axes to track across sources:
- **What is being optimized**: solutions, code/harnesses, system prompts, agent architecture, the optimization algorithm itself
- **Feedback signal**: scalar reward vs. rich diagnostic traces (execution logs, compiler errors, profiler output)
- **Loop structure**: single-agent self-edit vs. two-agent co-evolution vs. population-based evolution
- **Gating / safety**: regression thresholds, held-out eval sets, Pareto frontiers
- **Human involvement**: fully autonomous vs. human-in-the-loop review

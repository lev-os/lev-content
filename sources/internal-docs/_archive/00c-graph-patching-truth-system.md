# Graph Patching, Proposals & Truth System

> **Source:** `00-vision.md` lines 76-97, 334-353 (voice transcript)
> **Status:** classified → partially captured in `docs/design/graph-primitives.md`
> **Layer:** Site (core model — graph-runtime is foundational)
> **Interview notes:** User says entity ontology (Project→Workstream→Initiative etc.) is complementary to graph primitives — different actors for different parts. Graph primitives are for runtime; entity ontology is for project management / agentic workflow.

## Graph Patching

### Core Concept
Everything is a graph. The only way to change reality is to mutate the graph. A mutation is called a **Proposal** (or graph patch).

### Expand / Collapse
- When looking at a graph, you can **expand** (see more detail, simulate future steps) or **collapse** (simplify to essential nodes)
- A request can be a simple node → collapse the workflow
- Or complex → expand the simulation
- The LLM can project the next series of steps as a graph patch
- Each node is an agent; agents can cancel each other out ("that's not going to work anymore")
- "If you have 10 agents, you can probably traverse a graph all the way until this is done"

### Fractal Depth (3 Levels)
```
L0: Overview (status)
L1: Structure (zoom in, see component overviews)  
L2: Details (zoom in again, see implementation)
```
- Same structure at every level — fractal
- Content-dependent: image → metadata → where taken → keep going
- You can never have the full truth — always an approximation from a certain level
- "You have to hold all three levels of context across time and space all at the same time"

### Truth System
> "The truth is the graph. The graph is the truth."

- Truth docs (`##-thing.md` files) are **views** — not the source of truth themselves
- They are snapshots in time, ephemeral
- A proposal is just ephemeral reasoning — not a truth artifact
- "You could print out 100 documents to view it in as many granular ways as possible, or you can just view parts of the graph"
- Documents as views → rendered from beads/CMS content (see `02-views-are-bd-queries.md`)

### Relationship to Existing Docs
- `docs/design/graph-primitives.md` already defines: View, Claim, Evidence, TruthState, Proposal, Gate, Tick
- This transcript adds: expand/collapse metaphor, fractal depth rationale, truth-as-graph philosophy
- The "truth folder" is what became the `docs/` directory structure

## Key Insight (from interview)
> "Entities are all beads. BD is an API for a content management system. People use it as a task tracker, but it's really a CMS that will drive all of this. Markdown files are views based off the content and beads. You can query them and create websites — that's where the dashboard concept comes from."

## Open Questions
- [ ] How does expand/collapse map to FlowMind YAML? Is it a planning-time operation?
- [ ] What graph library to use? (mentioned as dependency type in transcript)
- [ ] How do proposals get materialized into the runtime graph?

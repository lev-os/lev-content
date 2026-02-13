# Dimension Model (PROPOSAL, Session 2026-02-12)

> Entity coordinate system for the Lev graph. Extends `graph-primitives.md` with an N-dimensional model where entities have extensible coordinates and Views are cross-sectional queries.

---

## Core Principle: Append-Only Graph

The graph is **append-only**. There is no undo.

- A "rollback" is a **new entry** that happens to produce the same code state as a prior entry — but it's a different timestamp, with different learnings, different evidence
- Git artifacts persist. Every mutation is recorded. History is the source of truth
- "Undo" exists *effectively* (you can reverse a change) but not *ontologically* (the reversal is its own event with its own context)

```
t0: entity created     → graph grows
t1: entity modified    → graph grows (old state preserved as history)
t2: entity "rolled back" → graph grows (new entry referencing t0, with t1's learnings attached)
```

The code at t0 and t2 may be identical. The graph is not — t2 carries the knowledge of *why* t1 failed.

---

## Dimension Registry

Entities have coordinates across N dimensions. Not all dimensions apply to all entities.

### Universal Dimensions (always present)

| Dimension | Type | Values | Question |
|-----------|------|--------|----------|
| **Depth** | ordinal | L0, L1, L2, L3 | How much detail am I seeing? |
| **Temporal** | continuous | timestamp, range, phase | When does/did/will this exist? |

### Structural Dimensions (present for systems, not atoms)

| Dimension | Type | Values | Question |
|-----------|------|--------|----------|
| **Layer** | ordinal | Site, Structure, Skin, Services, Space Plan, Stuff | What kind of thing is this? (permanence/nature) |

### Task-Specific Dimensions (attached when relevant)

| Dimension | Type | Values | Applies To |
|-----------|------|--------|-----------|
| **Risk** | ordinal | negligible → low → medium → high → critical | Planning, execution (not research/data) |
| **Complexity** | ordinal | trivial → simple → moderate → complex → epic | Planning, execution |
| **Effort** | continuous | minutes → hours → days → weeks → months | Planning, execution |
| **Confidence** | continuous | 0.0 → 1.0 | Claims, evidence, decisions |
| **Freshness** | continuous | stale → aging → current → live | Time-sensitive data |
| **Phase** | ordinal | concept → pre-release → alpha → beta → GA → EOL | Product lifecycle |
| **Spatial** | structured | geographic coords, network topology, deployment zone | Physical/distributed systems |
| **Ownership** | relational | person, team, org, agent | Governance, collaboration |

### Atom vs System

**Systems** have Layer coordinates (they have internal parts that change at different rates).

**Atoms** do not — they're leaf nodes:

| Atom Type | Has Depth? | Has Layer? | Has Temporal? | Example |
|-----------|-----------|-----------|--------------|---------|
| Scalar value | No | No | Yes | `42`, `true`, `"hello"` |
| Event | No | No | Yes | Earthquake, deploy, commit |
| Signal | No | No | Yes | "I feel anxious", error alert |
| Pure function | No | No | No | `add(a, b)` — timeless computation |

Atoms are the **leaf values** of the graph. Systems are the **nodes with internal structure**. Both participate in the graph but occupy different dimension subsets.

---

## Views as Cross-Sectional Queries

A View is a projection of the graph onto a subset of dimensions with filters on each:

```yaml
view:
  id: "view/architecture-overview"
  dimensions:
    depth: { min: "L0", max: "L1" }
    layer: ["Site", "Structure"]
    temporal: { as_of: "2026-02-12" }
    freshness: { min: "current" }
  format: "markdown"
  template: "architecture-overview.md.hbs"
```

### Projection Examples

**Schedule** = Temporal × Layer projection (Gantt chart):
```
dimension: temporal (x-axis) × layer (y-axis)
depth: L0 (labels only)

        Feb        Mar        Apr        May
Site    ├── SPEC ──┤
Struct  ├──────── IMPLEMENT ──────────────┤
Skin               ├──── UX DESIGN ──────┤
Services                    ├── INFRA ───┤
```

**Risk matrix** = Risk × Layer projection:
```
dimension: risk (x-axis) × layer (y-axis)
depth: L1 (structural components)

              low        medium       high       critical
Site                                              DB schema
Structure                             API contracts
Skin                     UI framework
Services      CI/CD      Monitoring
Space Plan    Features
Stuff         Config
```

**Ralph Loop** = Temporal × Tick projection (iteration sequence):
```
dimension: temporal (x-axis), with tick cycle (y-axis)

      Iteration 1       Iteration 2       Iteration 3
      ┌─────────┐       ┌─────────┐       ┌─────────┐
      │ INGEST  │       │ INGEST  │       │ INGEST  │
      │ OBSERVE │ graph │ OBSERVE │ graph │ OBSERVE │ graph
      │ PROPOSE │ state │ PROPOSE │ state │ PROPOSE │ state
      │ GATE    │       │ GATE    │       │ GATE    │
      │ APPLY   │ patch │ APPLY   │ patch │ APPLY   │ patch
      │ EMIT    │       │ EMIT    │       │ EMIT    │
      └────┬────┘       └────┬────┘       └────┬────┘
           │ context dies    │ context dies    │
           └────────────────►└────────────────►└──► done
             graph persists    graph persists

Each iteration reads graph state (append-only), proposes patches,
gates them, applies. Context resets. Graph grows.
Ralph Loop = Tick × N across context boundaries.
```

---

## Graph Patching and Future Projection

Mutations are always **Proposals** (from `graph-primitives.md`). The dimension model adds:

### Temporal Projection

An Expand operation (from graph-primitives) projects **future graph states** along the Temporal dimension:

```yaml
expand:
  from: "current-graph-state"
  temporal:
    horizon: "3 months"
    granularity: "weekly"
  projections:
    - proposal: "week-1-schema-migration"
      dimensions: { layer: "Site", depth: "L2", risk: "high" }
    - proposal: "week-3-api-update"
      dimensions: { layer: "Structure", depth: "L2", risk: "medium" }
    - proposal: "week-6-ui-refresh"
      dimensions: { layer: "Skin", depth: "L1", risk: "low" }
```

### History as Graph Growth

Every mutation adds to the graph. "Rollbacks" are forward-only:

```
Graph at t0:  [A] ──claims──► [X]
Graph at t1:  [A] ──claims──► [X']      (mutation applied)
Graph at t2:  [A] ──claims──► [X]       (same value as t0)
                   ──learned─► [Y]       (why X' didn't work)

t2 ≠ t0. Same code, different graph. t2 is richer — it carries t1's learnings.
```

### Snapshot Queries

Because the graph is append-only with timestamps, any historical state is queryable:

```yaml
# "What did the graph look like on Feb 1?"
view:
  temporal: { as_of: "2026-02-01" }
  depth: { max: "L1" }
  layer: ["*"]

# "What changed between Feb 1 and Feb 12?"
diff:
  temporal: { from: "2026-02-01", to: "2026-02-12" }
  layer: ["Site", "Structure"]  # only show foundational changes
```

---

## Relationship to Other Design Docs

| Doc | Relationship |
|-----|-------------|
| `graph-primitives.md` | Defines the primitives (View, Claim, Gate, etc.) that operate WITHIN this dimension space |
| `vernacular.md` | Defines terms for Layers (shearing) and Lifecycle States (temporal) |
| `actor-graph.md` | Defines actors that OCCUPY dimension coordinates |
| `00-process.md` | Execution Tier derived from Layer (risk + complexity) |

---

## Deep Research Queries

The following questions require deeper investigation to refine this model:

### 1. Existing Multi-Dimensional Graph Models in Industry

**Query**: What existing systems use N-dimensional coordinate models for knowledge graphs or entity management? Specifically: how do systems like Notion databases, Roam Research, or enterprise knowledge graphs (Neo4j, TigerGraph) handle extensible property dimensions vs fixed schemas? How do OLAP cubes, data warehouses, and analytics engines (Druid, ClickHouse, Pinot) handle time-series + categorical dimension queries — and can those patterns apply to an append-only entity graph?

### 2. Temporal Graph Databases and Append-Only Event Sourcing

**Query**: How do temporal graph databases (e.g., TerminusDB, TypeDB, Datomic) implement point-in-time queries on append-only graphs? What are the storage and query performance tradeoffs vs mutable graph stores? How does event sourcing (Axon Framework, EventStoreDB) handle "effective rollback" (compensating events) vs "actual rollback" — and which pattern maps best to our "rollback is a new entry with learnings" model?

### 3. Spatial-Temporal Visualization of Agent Execution Loops

**Query**: How do existing agent observability tools (LangSmith, Langfuse, Arize Phoenix, Weights & Biases Traces) visualize iterative agent loops (Ralph Loops, ReAct chains, tool-use sequences)? What visualization patterns exist for showing context reset boundaries, state persistence across iterations, and graph growth over time? Are there precedents for combining Gantt-style temporal views with graph-topology views in a single dashboard (e.g., Grafana + Neo4j, Kibana + graph viz)?

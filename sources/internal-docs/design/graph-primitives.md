# Graph Primitives and Depth Model (LOCKED)

> Extracted from `docs/01-architecture.md`. This file describes the target graph-runtime model. Current canonical design authority is `~/lev/docs`.

---

## Multi-Axis Depth Model (LOCKED, Session 2026-02-12)

Entities in the graph have coordinates across two orthogonal axes. Views are cross-sectional queries across both.

### Axis 1: Depth (L0-L3) — Zoom / Level of Detail

```
L0: OVERVIEW      — High-level summary, orientation, "what is this?"
L1: STRUCTURE     — Components, boundaries, relationships
L2: DETAILS       — Implementation, specifics, how things work
L3: RUNTIME       — Behavior, state transitions, live data
```

**Fractal property**: Same structure at every level. You can always zoom in (expand) or zoom out (collapse). You can never have the full truth — always an approximation from a certain level. "You have to hold all three levels of context across time and space all at the same time."

### Axis 2: Layer (6 Shearing Layers) — Shape / Nature

Adapted from Stewart Brand's "How Buildings Learn." Describes the **nature and permanence** of what you're looking at. Applies to any domain — software, sports, entertainment, organizations.

| Layer | Timescale | Building | General |
|-------|-----------|----------|---------|
| **Site** | Decades+ | The land | Core identity, foundational model, what defines the thing |
| **Structure** | 30-50y | Load-bearing walls | Contracts, schemas, architecture, load-bearing decisions |
| **Skin** | ~20y | Exterior surface | Interface, brand, how it presents to the world |
| **Services** | 7-15y | Plumbing/HVAC | Infrastructure, operations, what runs behind the scenes |
| **Space Plan** | 3-7y | Interior layout | Features, capabilities, what changes without touching structure |
| **Stuff** | Daily | Furniture | Config, content, daily ephemera |

**Names are generic.** Each domain applies its own labels via plugin config. What matters is the timescale/permanence gradient.

### Multi-Axis Coordinate

Every entity occupies a position on both axes:

```yaml
entity:
  layer: "Structure"    # what kind of thing (shape/nature)
  depth: "L2"           # how much detail you're seeing (zoom)
```

Views are cross-sectional queries:

```
# "Overview down to details for foundational things"
graph.query({ layers: ["Site", "Structure"], depth: { min: "L0", max: "L2" } })

# "Surface-level snapshot of everything"
graph.query({ layers: ["*"], depth: { min: "L0", max: "L0" } })

# "Runtime details for features only"
graph.query({ layers: ["Space Plan"], depth: { min: "L3", max: "L3" } })
```

### Derived Dimensions

**Risk** = `layer.timescale × depth.specificity`. Site+L3 = maximum risk. Stuff+L0 = negligible.

**Complexity** = gauged per entity or derived from graph density (claim count, evidence links, dependency fan-out).

**Execution Tier** (from `00-process.md`): Site/Structure → T1 Formal. Skin/Services → T2-T3. Space Plan/Stuff → T3-T4 Direct.

### Example: Lev (Software)

| Layer | Label | L0 | L1 | L2 | L3 |
|-------|-------|----|----|----|----|
| Site | Core Model | "Universal agent runtime" | Graph primitives, event log, depth model | Claim schema YAML, LevEvent fields | Live event replay, MVCC state |
| Structure | Architecture | Hex arch overview | Module boundaries, contracts | `IHarness`, `IProvider` interfaces | Contract test results |
| Skin | Interface | CLI + TUI + dashboard | `lev` command surface, TUI layout | Component props, keyboard bindings | Active TUI session state |
| Services | Infrastructure | CI/CD + search + daemons | Polyglot search engines, daemon registry | LEANN index config, heartbeat spec | Index freshness, daemon PIDs |
| Space Plan | Features | FlowMind, plugins, skills | Workflow YAML schema, plugin manifest | Compiler IR, assembler targets | Running workflow trace |
| Stuff | Config | "Config is fractal" | XDG paths, env var list | `config.yaml` schema, merge semantics | Current resolved config values |

### Example: Baseball Team

| Layer | Label | L0 | L1 | L2 | L3 |
|-------|-------|----|----|----|----|
| Site | Identity | "The New York Yankees" | AL East, 27 titles, Yankee Stadium | Revenue model, TV deal structure | YoY revenue, brand sentiment |
| Structure | Organization | Front office + farm system | GM, coaching hierarchy, minor league pipeline | Scouting criteria, draft strategy | Active trade negotiations |
| Skin | Brand | Pinstripes, legacy | Stadium experience, broadcast identity | Merchandise design, social voice | Fan engagement metrics |
| Services | Operations | Scouting + analytics | Analytics stack, medical staff protocol | Statcast data pipeline, rehab process | Live Statcast feed |
| Space Plan | Roster | Active 26-man roster | Lineup decisions, rotation, bullpen roles | Player stat splits, platoon matchups | In-game substitution state |
| Stuff | Daily | Today's game | Lineup card, weather, injury report | Pitch selection tendencies, defensive shifts | Pitch-by-pitch, live score |

### Example: Pre-Release Movie (temporal element)

| Layer | Label | L0 | L1 | L2 | L3 |
|-------|-------|----|----|----|----|
| Site | IP/Story | "Blade Runner 2099" | Source material, genre, universe | Core narrative arc, thematic intent | *(unreleased — no runtime)* |
| Structure | Production | Director, studio, budget | Timeline, distribution deal, cast | Shooting schedule, VFX contracts | *(wrapping — daily call sheets)* |
| Skin | Marketing | Trailer dropped, poster out | Press tour schedule, brand partnerships | Trailer cut decisions, poster rationale | Live trailer views, social buzz |
| Services | Distribution | Theatrical + streaming | Theater chain deals, international dates | Screen count by market, streaming window | *(pre-release — no runtime)* |
| Space Plan | Content | "8 episodes, noir sci-fi" | Episode breakdown, character arcs | Scene-level VFX, soundtrack cues | *(post-production — dailies only)* |
| Stuff | Buzz | "Most anticipated 2026" | Pre-sale numbers, screening dates | Review embargo terms, mention volume | Live pre-sale velocity, trending rank |

**Temporal property**: Not all entities have data at all depths yet. Pre-release content has gaps at L3 — that's a genuine property of the model, not missing data.

### Plugin Extensibility

Plugins define their own Layer × Depth semantics via config:

```yaml
# plugins/baseball/config.yaml
layers:
  Site: { label: "Identity", entities: [franchise, league, stadium] }
  Structure: { label: "Organization", entities: [front_office, farm_system] }
  Skin: { label: "Brand", entities: [uniform, broadcast, social] }
  Services: { label: "Operations", entities: [scouting, analytics, medical] }
  Space Plan: { label: "Roster", entities: [player, lineup, rotation] }
  Stuff: { label: "Daily", entities: [game, lineup_card, injury_report] }
depth:
  L0: { label: "Overview" }
  L1: { label: "Organization" }
  L2: { label: "Stats" }
  L3: { label: "Live" }
```

---

## Graph Primitives (LOCKED)

### View

A **View** is a materialized projection over graph state — lossy by design, scoped and time-bounded.

```yaml
view:
  id: "view/01-architecture"
  target: "lev-os"
  type: "ArchitectureOverview"
  query:
    include: ["modules", "principles", "vernacular"]
    depth: 2  # L0-L1
  freshness:
    desired_ms: 86400000  # daily
```

**Future-state note:** graph-backed stores may materialize Views, but until that runtime exists, `~/lev/docs` is the canonical design source.

### Claim

An **atomic assertion** about entities under scope/time. Claims have TruthState.

```yaml
claim:
  id: "claim/flowmind-declares-triggers-executes"
  subject: "lev-os/core"
  predicate: "architectural_principle"
  object: "FlowMind DECLARES, Triggers EXECUTES, Events is SPINE"
  truth_state: "accepted"  # accepted|rejected|unknown|disputed|stale
  evidence_refs: ["evidence/session-6-consensus"]
```

### Evidence

**Support for claims** (observation, log, measurement, user attestation, derived).

```yaml
evidence:
  id: "evidence/session-6-consensus"
  kind: "user_attest"
  source: "session/20260203-events-flowmind-kernel"
  payload_ref: "handoffs/20260203-events-flowmind-kernel.md"
  reliability: 0.97
```

### TruthState

System stance on a claim under scope/time:
- `accepted` — Claim validated by evidence
- `rejected` — Claim invalidated by evidence
- `unknown` — Insufficient evidence
- `disputed` — Conflicting evidence
- `stale` — Evidence outdated

### Proposal

A **candidate graph mutation** — operations list requiring gates.

```yaml
proposal:
  id: "proposal/observe-to-events-rename"
  intent: "Improve discoverability"
  operations:
    - op: "rename_entity"
      from: "lev-os/core/observe"
      to: "lev-os/core/events"
  gates_required: ["Gate/Scope", "Gate/Standards"]
  status: "pending"  # pending|amended|accepted|rejected
```

### Expand / Collapse

**Planning-time operation over the graph.** When looking at a graph:

- **Expand**: See more detail, simulate future steps as a series of candidate graph patches
- **Collapse**: Simplify to essential nodes when the request is simple enough

The LLM projects the next series of steps as a graph patch (Proposal). Each node can be an agent; agents can cancel each other's proposals if they conflict.

```yaml
expand:
  from: "current-graph-state"
  depth: 3  # How many steps to project forward
  mode: "simulate"  # simulate|plan|execute
  result:
    - proposal: "step-1"  # Each step is a Proposal
    - proposal: "step-2"
    - proposal: "step-3"
  collapse_if: "simple"  # Collapse entire workflow if request is trivial
```

Relationship: Expand/Collapse operates on Proposals and Views. Expanding generates Proposals; collapsing skips intermediate Proposals.

### Interface vs ActionSurface

| Primitive | Definition | Examples |
|-----------|------------|----------|
| **Interface** | Continuous interaction medium | Conversation, VR presence, dashboard |
| **ActionSurface** | Discrete intent→action boundary | Button, CLI command, gesture, hotkey |
| **ActionExtractor** | Converts Interface streams → ActionSurface events | Confidence thresholds, confirmations |

### Gate

**Evaluator that allows/amends/rejects proposals by enforcing Leases:**

| Gate | Purpose | Enforces |
|------|---------|----------|
| Gate/Scope | Is this within project/workstream boundary? | Scope lease |
| Gate/Safety | Secrets, destructive actions? | Safety lease |
| Gate/Feasibility | Tools available? Permissions? | Feasibility lease |
| Gate/Standards | Rules, formatting, invariants? | Standards lease |
| Gate/Progress | Did last tick measurably advance claims? | Progress lease |

### Lease

**A rule/contract an agent must fulfill before proceeding.** Gates enforce leases. Brand name: AgentLease.

```yaml
lease:
  id: "lease/must-pass-tests"
  type: "pre-commit"
  rule: "All tests must pass before commit is allowed"
  enforced_by: ["Gate/Standards"]
  surface: "git"  # ActionSurface where this applies
  status: "active"  # active|expired|fulfilled|violated
```

Relationship: `ActionSurface → Interface → Gate → Lease`
- Gate is the checkpoint (enforcer)
- Lease is the contract (what's enforced)
- The only way to enforce a lease is with a gate

### Tick

**Scheduler heartbeat** — the loop that drives progress:

```
1. INGEST     — collect ActionSurface events
2. OBSERVE    — read current graph state
3. PROPOSE    — agents + heuristics generate proposals
4. GATE       — run proposals through Gates
5. APPLY      — accepted operations mutate graph
6. UPDATE     — refresh affected Views
7. EMIT       — publish LevEvents to Bus
```

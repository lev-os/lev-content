# Leviathan: Actor-Graph Architecture

**Status**: DESIGN PROPOSAL
**Created**: 2026-02-04
**Author**: Session synthesis (Vigil intake → Actor model exploration)

---

## Executive Summary

Two graphs, two purposes:

1. **System Graph** - The proposed persistent state model of Leviathan. `~/lev/docs` is the tightly controlled system-design view for this model.
2. **CDO Graphs** - Ephemeral thinking graphs. Created, used, discarded.

**Core insight**: `~/lev/docs` is a tightly controlled, coherent system-design view of the graph model. CDO creates temporary thinking graphs to produce meaningful content, then discards them.

---

## The Two Graphs

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         TWO GRAPHS, TWO PURPOSES                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────────────────────────┐  ┌────────────────────────────┐ │
│  │       SYSTEM GRAPH (persistent)    │  │   CDO GRAPH (ephemeral)    │ │
│  │                                     │  │                            │ │
│  │  What Leviathan IS                  │  │  Thinking workspace        │ │
│  │  - Modules, dependencies            │  │  - Agents (actors)         │ │
│  │  - Architecture decisions           │  │  - Turns, sync points      │ │
│  │  - Locked contracts                 │  │  - Artifacts (files)       │ │
│  │                                     │  │                            │ │
│  │  ~/lev/docs = System-design VIEW       │  │  Created → Used → Discarded│ │
│  │  (tightly controlled, coherent docs)   │  │  Only OUTPUT persists      │ │
│  │                                     │  │                            │ │
│  └───────────────────────────────────┘  └────────────────────────────┘ │
│                │                                       │                │
│                │                                       │                │
│                │         ┌─────────────────┐          │                │
│                └────────▶│  FINAL.md       │◀─────────┘                │
│                          │  (meaningful    │                           │
│                          │   content)      │                           │
│                          └────────┬────────┘                           │
│                                   │                                     │
│                                   ▼                                     │
│                          ┌─────────────────┐                           │
│                          │  Event Log      │                           │
│                          │  (audit trail)  │                           │
│                          └─────────────────┘                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### System Graph

- **Persistent** - Lives forever
- **~/lev/docs is a VIEW** - Just markdown files representing current state
- **Contains**: Modules, architecture, decisions, contracts
- **Changes via**: Graph patches (audited, reversible)

### CDO Graph

- **Ephemeral** - Created for a workflow, discarded after
- **Agents are actors** - state + commands + queries + broadcast
- **Isolation** - Agents don't see each other during execution
- **Output** - Only FINAL.md (or equivalent) persists
- **Purpose** - Produce meaningful content through structured thinking

---

## Part 1: Agents as Actors

### The Actor Contract

Every agent in Leviathan implements the Actor interface:

```typescript
interface LevActor<TState, TCommands, TQueries> {
  // Isolated state - only this actor can modify
  state: TState;

  // Commands - async operations that change state + emit events
  commands: TCommands;

  // Queries - sync reads of current state
  queries: TQueries;

  // Broadcast - notify graph without coupling
  broadcast(event: string, payload: unknown): void;
}
```

### Why Actors?

| Property | Benefit for Agents |
|----------|-------------------|
| **Isolation** | Agents can't corrupt each other's state |
| **Commands** | State changes are explicit, trackable |
| **Queries** | Other actors read without side effects |
| **Broadcast** | Loose coupling, no direct dependencies |

### Agent Actor Example

```typescript
export const ideaAgent = actor<IdeaState>({
  state: {
    id: null,
    content: null,
    stage: "captured",
    confidence: 0,
    relatedTo: [],
  },

  commands: {
    capture: async (c, input: string) => {
      c.state.id = generateId();
      c.state.content = input;
      c.state.stage = "captured";

      // Emit event to audit trail
      c.broadcast("idea.captured", {
        id: c.state.id,
        content: input,
        timestamp: Date.now()
      });

      return c.state.id;
    },

    crystallize: async (c) => {
      // CDO orchestration happens here
      const analysis = await runCDOWorkflow(c.state.content);
      c.state.confidence = analysis.confidence;
      c.state.stage = "crystallized";

      c.broadcast("idea.crystallized", {
        id: c.state.id,
        confidence: c.state.confidence
      });
    },

    manifest: async (c, plan: ExecutionPlan) => {
      // Execute the plan
      c.state.stage = "manifesting";
      c.broadcast("idea.manifesting", { id: c.state.id, plan });

      // ... execution ...

      c.state.stage = "manifested";
      c.broadcast("idea.manifested", { id: c.state.id });
    }
  },

  queries: {
    getState: (c) => c.state,
    getConfidence: (c) => c.state.confidence,
    canTransition: (c, to: string) => {
      // State machine validation
      return IDEA_TRANSITIONS[c.state.stage]?.includes(to);
    }
  }
});
```

---

## Part 2: The Graph Structure

### Nodes (Actors)

| Node Type | State | Commands | Broadcasts |
|-----------|-------|----------|------------|
| **IdeaActor** | content, stage, confidence | capture, crystallize, manifest | idea.* |
| **SessionActor** | artifacts, ideaIds, status | start, pause, complete | session.* |
| **AgentActor** | config, health, current_task | configure, execute, report | agent.* |
| **ModuleActor** | spec, dependencies, status | refine, validate, lock | module.* |
| **TruthActor** | documents, version, locked | update, lock_decision | truth.* |

### Edges (Relationships)

```typescript
type GraphEdge =
  | { type: "WORKS_ON", from: SessionActor, to: IdeaActor }
  | { type: "DEPENDS_ON", from: ModuleActor, to: ModuleActor }
  | { type: "SPAWNED_BY", from: AgentActor, to: AgentActor }
  | { type: "DOCUMENTS", from: TruthActor, to: ModuleActor }
  | { type: "LISTENS_TO", from: Actor, to: Actor }
```

### Graph Patches

Changes to the graph are expressed as patches:

```typescript
interface GraphPatch {
  id: string;
  timestamp: number;
  author: string;  // Agent that made the change
  operations: PatchOperation[];
}

type PatchOperation =
  | { op: "add_node", node: Actor }
  | { op: "remove_node", nodeId: string }
  | { op: "add_edge", edge: GraphEdge }
  | { op: "remove_edge", edgeId: string }
  | { op: "update_state", nodeId: string, path: string, value: unknown }
  | { op: "broadcast", event: string, payload: unknown }
```

### Example: Agent Creates Idea

```typescript
const patch: GraphPatch = {
  id: "patch-001",
  timestamp: Date.now(),
  author: "agent:researcher",
  operations: [
    // Create idea node
    { op: "add_node", node: ideaActor({ id: "idea-42", content: "..." }) },

    // Link to session
    { op: "add_edge", edge: {
      type: "WORKS_ON",
      from: "session:current",
      to: "idea:idea-42"
    }},

    // Broadcast event
    { op: "broadcast", event: "idea.captured", payload: { id: "idea-42" } }
  ]
};

// Apply patch → emits events → projects to ~/lev/docs
await graph.applyPatch(patch);
```

---

## Part 3: CDO = Ephemeral Graph Factory

CDO creates temporary graphs for thinking:

```
┌──────────────────────────────────────────────────────────────────┐
│                    CDO = Graph Orchestration                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│   QUICK (1-2 agents, 1 turn)                                     │
│   ┌─────┐                                                        │
│   │  A  │──────▶ FINAL.md                                        │
│   └─────┘                                                        │
│                                                                   │
│   BASE (2-3 agents, 2 turns)                                     │
│   ┌─────┐                                                        │
│   │  A  │──┐                                                     │
│   └─────┘  │     ┌─────┐                                         │
│            ├────▶│  C  │──────▶ FINAL.md                         │
│   ┌─────┐  │     └─────┘                                         │
│   │  B  │──┘     (sync)                                          │
│   └─────┘                                                        │
│                                                                   │
│   DEEP (3-5 agents, 3-5 turns)                                   │
│   ┌─────┐     ┌─────┐     ┌─────┐                                │
│   │  A  │────▶│  B  │────▶│  C  │──┐                             │
│   └─────┘     └─────┘     └─────┘  │     ┌─────┐                 │
│                                     ├────▶│ SYN │──▶ FINAL.md    │
│   ┌─────┐     ┌─────┐     ┌─────┐  │     └─────┘                 │
│   │  D  │────▶│  E  │────▶│  F  │──┘                             │
│   └─────┘     └─────┘     └─────┘                                │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### CDO Graph Semantics

```typescript
interface CDOGraph {
  // Nodes are agent actors
  agents: Map<string, AgentActor>;

  // Edges define execution flow
  edges: {
    type: "SEQUENTIAL" | "PARALLEL" | "SYNC_POINT" | "BROADCAST";
    from: string;
    to: string;
    condition?: (state: GraphState) => boolean;
  }[];

  // Turn = sync point for parallel agents
  turns: Turn[];
}

interface Turn {
  id: number;
  agents: string[];  // Agents executing in parallel
  syncPoint: () => Promise<void>;  // Wait for all to complete
  artifacts: string[];  // Files written this turn
}
```

### CDO as Graph Patch Sequence

```typescript
// A CDO workflow is a sequence of patches
const cdoWorkflow: GraphPatch[] = [
  // Turn 1: Spawn parallel agents
  {
    operations: [
      { op: "add_node", node: agentA },
      { op: "add_node", node: agentB },
      { op: "add_edge", edge: { type: "PARALLEL", from: "turn:1", to: "agent:a" } },
      { op: "add_edge", edge: { type: "PARALLEL", from: "turn:1", to: "agent:b" } },
    ]
  },

  // Turn 1 complete: Sync point
  {
    operations: [
      { op: "broadcast", event: "turn.complete", payload: { turn: 1 } },
      { op: "update_state", nodeId: "agent:a", path: "status", value: "done" },
      { op: "update_state", nodeId: "agent:b", path: "status", value: "done" },
    ]
  },

  // Turn 2: Synthesis agent reads all artifacts
  {
    operations: [
      { op: "add_node", node: synthAgent },
      { op: "add_edge", edge: { type: "READS", from: "agent:synth", to: "agent:a" } },
      { op: "add_edge", edge: { type: "READS", from: "agent:synth", to: "agent:b" } },
    ]
  },

  // Turn 2 complete: Final output
  {
    operations: [
      { op: "broadcast", event: "cdo.complete", payload: { output: "FINAL.md" } },
    ]
  }
];
```

---

## Part 4: ~/lev/docs as Materialized View

### The Key Insight

**~/lev/docs is NOT the source of truth. The event log is.**

```
Events (authoritative)  →  Project  →  ~/lev/docs (view)
```

### Truth Actor

```typescript
export const truthActor = actor<TruthState>({
  state: {
    documents: {},     // module -> document content
    version: 0,
    lockedDecisions: [],
    refinementPass: "L1"
  },

  commands: {
    // Rebuild from event log
    rebuild: async (c) => {
      const events = await eventLog.readAll();
      c.state = projectToTruth(events);
      c.broadcast("truth.rebuilt", { version: c.state.version });
    },

    // Apply incremental update from new event
    applyEvent: (c, event: DomainEvent) => {
      const patch = eventToTruthPatch(event);
      c.state = applyPatch(c.state, patch);
      c.state.version++;

      // Write to filesystem
      writeDocument(patch.affectedDoc, c.state.documents[patch.affectedDoc]);

      c.broadcast("truth.updated", {
        version: c.state.version,
        affectedDoc: patch.affectedDoc
      });
    },

    // Lock architectural decision
    lockDecision: (c, decision: string) => {
      c.state.lockedDecisions.push(decision);
      c.broadcast("truth.decision_locked", { decision });
    },

    // Advance refinement pass
    advancePass: (c) => {
      const passes = ["L1", "L2", "L3"];
      const currentIndex = passes.indexOf(c.state.refinementPass);
      if (currentIndex < passes.length - 1) {
        c.state.refinementPass = passes[currentIndex + 1];
        c.broadcast("truth.pass_advanced", { pass: c.state.refinementPass });
      }
    }
  },

  queries: {
    getDocument: (c, name: string) => c.state.documents[name],
    isLocked: (c, decision: string) => c.state.lockedDecisions.includes(decision),
    getCurrentPass: (c) => c.state.refinementPass,
    getVersion: (c) => c.state.version
  }
});
```

### Event → Truth Projection

```typescript
function projectToTruth(events: DomainEvent[]): TruthState {
  const state: TruthState = {
    documents: {},
    version: events.length,
    lockedDecisions: [],
    refinementPass: "L1"
  };

  for (const event of events) {
    switch (event.type) {
      case "MODULE_SPEC_UPDATED":
        state.documents[event.payload.module] = event.payload.spec;
        break;

      case "DECISION_LOCKED":
        state.lockedDecisions.push(event.payload.decision);
        break;

      case "REFINEMENT_PASS_ADVANCED":
        state.refinementPass = event.payload.pass;
        break;

      case "ARCHITECTURE_CHANGED":
        state.documents["01-architecture.md"] = event.payload.content;
        break;
    }
  }

  return state;
}
```

### Addressing Documentation Sprawl

```
PROBLEM: Documentation sprawl
─────────────────────────────
- 50+ markdown files across repos
- No single source of truth
- Docs contradict each other
- Can't tell what's current

SOLUTION: ~/lev/docs as materialized view
───────────────────────────────────────
- Event log is authoritative
- ~/lev/docs is COMPUTED from events
- Can always rebuild from scratch
- Agents query truth actor, not files
- Changes go through commands (audited)
```

---

## Part 5: The Audit Trail

### Every Change is an Event

```typescript
interface AuditEvent {
  id: string;
  timestamp: number;

  // Who
  actor: {
    type: "agent" | "user" | "system";
    id: string;
  };

  // What
  event: string;
  payload: unknown;

  // Tracing
  correlationId?: string;  // Group related events
  causationId?: string;    // What caused this event

  // Graph context
  affectedNodes: string[];
  affectedEdges: string[];
}
```

### Audit Trail Use Cases

| Use Case | Query |
|----------|-------|
| What changed? | `events.where(timestamp > X)` |
| Who changed it? | `events.where(actor.id == "agent:Y")` |
| Why did it change? | `events.where(causationId == Z)` → trace chain |
| Can I undo? | `events.where(affectedNodes.includes(N))` → reverse patches |
| What was truth at time T? | `projectToTruth(events.where(timestamp <= T))` |

### JSONL Storage

```jsonl
{"id":"ev-001","ts":1707033600,"actor":{"type":"agent","id":"researcher"},"event":"idea.captured","payload":{"id":"idea-42","content":"..."}}
{"id":"ev-002","ts":1707033605,"actor":{"type":"agent","id":"researcher"},"event":"idea.crystallized","payload":{"id":"idea-42","confidence":0.85},"causationId":"ev-001"}
{"id":"ev-003","ts":1707033610,"actor":{"type":"system","id":"truth"},"event":"truth.updated","payload":{"version":3,"affectedDoc":"04-core-index.md"},"causationId":"ev-002"}
```

---

## Part 6: Agentic CMS

### Agents Manage Content

```typescript
// Content is managed through agent actors
const cmsGraph = {
  // Document actors (one per document in ~/lev/docs)
  documents: {
    "01-architecture": documentActor({ path: "01-architecture.md" }),
    "04-core-index": documentActor({ path: "04-core-index.md" }),
    // ...
  },

  // Agent actors that modify documents
  agents: {
    "refiner": refinerAgent(),      // Applies refinement passes
    "validator": validatorAgent(),   // Validates contracts
    "integrator": integratorAgent(), // Ensures cross-references
  },

  // Truth actor aggregates all documents
  truth: truthActor()
};
```

### Document Actor

```typescript
export const documentActor = (config: { path: string }) => actor<DocState>({
  state: {
    path: config.path,
    content: null,
    version: 0,
    lastModifiedBy: null,
    lockedSections: []
  },

  commands: {
    update: async (c, content: string, author: string) => {
      c.state.content = content;
      c.state.version++;
      c.state.lastModifiedBy = author;

      c.broadcast("document.updated", {
        path: c.state.path,
        version: c.state.version,
        author
      });
    },

    lockSection: (c, section: string) => {
      c.state.lockedSections.push(section);
      c.broadcast("document.section_locked", { path: c.state.path, section });
    }
  },

  queries: {
    getContent: (c) => c.state.content,
    getVersion: (c) => c.state.version,
    isLocked: (c, section: string) => c.state.lockedSections.includes(section)
  }
});
```

### Agent-Document Workflow

```
User: "Update the index module spec"
                │
                ▼
        ┌───────────────┐
        │ Refiner Agent │  (commands.update)
        └───────┬───────┘
                │
                ▼
        ┌───────────────┐
        │ Doc: 04-core  │  (state changes)
        │    -index.md  │
        └───────┬───────┘
                │ broadcast("document.updated")
                ▼
        ┌───────────────┐
        │ Truth Actor   │  (listens, rebuilds)
        └───────┬───────┘
                │ broadcast("truth.updated")
                ▼
        ┌───────────────┐
        │ Event Log     │  (persists)
        └───────────────┘
```

---

## Implementation Roadmap

### Phase 1: Actor Primitives (Week 1)

```bash
# Create core/actors/ module
mkdir -p ~/lev/core/actors/src

# Implement base actor
touch ~/lev/core/actors/src/actor.ts
touch ~/lev/core/actors/src/graph.ts
touch ~/lev/core/actors/src/patch.ts
```

Deliverables:
- [ ] `Actor<TState, TCommands, TQueries>` interface
- [ ] `GraphPatch` type and apply function
- [ ] `broadcast()` mechanism via event bus

### Phase 2: Truth Actor (Week 2)

```bash
# Create truth actor
touch ~/lev/core/actors/src/truth-actor.ts

# Projection from events
touch ~/lev/core/actors/src/projections/truth.ts
```

Deliverables:
- [ ] `TruthActor` with rebuild/applyEvent commands
- [ ] Event → Truth projection function
- [ ] Sync to filesystem on update

### Phase 3: CDO Integration (Week 3)

```bash
# Refactor CDO to use actor graph
# Update core/cdo/
touch ~/lev/core/cdo/src/graph-executor.ts
```

Deliverables:
- [ ] CDO workflows as graph patches
- [ ] Agent actors with file I/O isolation
- [ ] Turn-based sync points

### Phase 4: Agentic CMS (Week 4)

```bash
# Document actors
touch ~/lev/core/actors/src/document-actor.ts

# Agent actors
touch ~/lev/core/actors/src/agents/refiner.ts
touch ~/lev/core/actors/src/agents/validator.ts
```

Deliverables:
- [ ] Document actors for each ~/lev/docs file
- [ ] Refiner, Validator, Integrator agents
- [ ] Full audit trail in `.lev/events.jsonl`

---

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Source of truth | Event log | Immutable, auditable, rebuildable |
| ~/lev/docs role | Materialized view | Fast queries, can rebuild |
| Agent isolation | Actor model | No shared state, explicit communication |
| Changes | Graph patches | Atomic, reversible, auditable |
| CDO execution | Graph traversal | Natural fit for multi-agent workflows |

---

## Relates

This document connects:
- **01-architecture.md** - Overall Lev architecture
- **lev-cdo skill** - CDO workflow patterns
- **core/events/** - Event sourcing infrastructure
- **Vigil intake** - Actor model inspiration

---

**Next**: Review and approve for implementation.

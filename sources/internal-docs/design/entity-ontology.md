# Entity Ontology — SDLC Plugin Layer (PROPOSAL)

> Extracted from `docs/_inbox/00e-ux-stages-entity-ontology.md`. These entities are **SDLC plugin entities**, NOT core graph primitives. They compose on top of the graph-primitive layer (Entity/Claim/Evidence/Proposal/View/Gate/Lease/Tick).

---

## Layer Separation

```
Layer 0: Graph Primitives (core kernel)     — Entity, Claim, Evidence, Proposal, View, Gate, Lease, Tick
Layer 1: SDLC Plugin (domain entities)      — Project, Workstream, Initiative, Agent, Session, Daemon, Skill
Layer 2: Dashboard (presentation)           — URI scheme, component intent, FSM
```

**Core operates against an abstract CMS interface.** Beads (BD) is one adapter. Others: BR, TD, WordPress, SQLite, Notion, Obsidian.

---

## Entity Hierarchy

| Entity | Parent | Children | CMS Content Type |
|--------|--------|----------|-----------------|
| **Project** | — | Workstreams, Daemons, Skills, Config | `project` |
| **Workstream** | Project | Initiatives, Agents | `workstream` |
| **Initiative** | Workstream | Tasks (BD issues) | `initiative` / `epic` |
| **Agent** | Workstream | Sessions | `agent` |
| **Session** | Agent | Messages, Artifacts | `session` |
| **Daemon** | Project | Processes, Ports | `daemon` |
| **Skill** | Project | Rules, Templates | `skill` |

All entities are **beads** in the CMS. Markdown files are **views** rendered from CMS content.

---

## Entity Actions

| Action | Target | Result | Graph Operation |
|--------|--------|--------|----------------|
| spawn | Agent | New session created | `add_entity(session)` + `add_link(agent, session)` |
| delegate | Workstream | Assigned to agent/swarm | `add_link(workstream, agent)` |
| pause/resume | Session | State preserved | `patch_entity(session, {status})` |
| configure | Project/Agent | Settings updated | `patch_entity(target, {config})` |
| switch | Project | All panels update | UI-only (no graph mutation) |
| collapse/expand | Panel | UI state change | UI-only |

---

## URI Scheme

```
project/leviathan        → Project overview
workstream/auth-refactor → Workstream board
initiative/lev-r9no      → Epic detail (BD)
agent/session:abc-123    → Session view
agent/config:worker-1    → Agent config
agent/new                → Spawn wizard
daemon/dev               → Daemon detail
```

---

## CMS Adapter Interface

```typescript
interface CmsAdapter {
  // CRUD
  create(type: string, data: Record<string, unknown>): Promise<string>;
  read(id: string): Promise<Entity>;
  update(id: string, patch: Record<string, unknown>): Promise<void>;
  delete(id: string): Promise<void>;

  // Query
  list(type: string, filters?: Record<string, unknown>): Promise<Entity[]>;
  query(q: string): Promise<Entity[]>;  // Adapter-specific query language

  // View rendering
  render(template: string, entities: Entity[]): string;
  export(format: 'markdown' | 'json' | 'obsidian', query: string): string;
}
```

Adapters must support custom content types for all entities in the hierarchy.

---

## Relationship to Actor-Graph

| Actor-Graph Actor | SDLC Entity | Notes |
|-------------------|-------------|-------|
| IdeaActor | (none — exists at graph level) | Ideas → entities via lifecycle |
| SessionActor | Session | Same concept |
| AgentActor | Agent | Same concept |
| ModuleActor | (none — code-level) | Below SDLC layer |
| TruthActor | (none — system-level) | Manages views |

The actor-graph actors are **runtime behaviors**. SDLC entities are **content types** in the CMS. An AgentActor *manages* an Agent entity.

---

**Status**: PROPOSAL — needs review before locking.

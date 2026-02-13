# Dashboard UX Spec — Multi-Project Agent Dashboard (PROPOSAL)

> Extracted from `docs/_inbox/00e-ux-stages-entity-ontology.md`. Follows the 6-stage UX research process.
> Depends on: `entity-ontology.md`, `graph-primitives.md`, `vernacular.md`

---

## Stage 1: Problem Framing

**Problem:** When running multiple concurrent AI agent projects, the mental model collapses. Work is split between apps (terminal, browser, IDE) with no natural grouping.

**Success Criteria:**

| Metric | Target |
|--------|--------|
| Notification → project identification | <1s |
| Port collisions | zero |
| Context switch cost | 1 click |
| Agent/session scoping | per-project |
| User sentiment | "I know where everything is" |

**Target Users:**
- Developers running 2-5 AI agent projects simultaneously
- Power users comfortable with CLI but need visual overview
- Team leads managing swarm delegation across workstreams

---

## Stage 2: Jobs-to-Be-Done

| Job | Trigger | Outcome |
|-----|---------|---------|
| Know where I am | Notification | Identify project + agent in <1s |
| Switch contexts | "Check Project B" | All panels update to B's state |
| Spawn work | New feature/bug | Delegate to agent with scope |
| Monitor progress | Periodic check | See all agent states at glance |
| Resolve conflicts | Port collision | System prevents or auto-resolves |
| Resume session | Return from break | UI shows where I left off |
| Delegate portfolio | Too much in flight | Assign workstreams to sub-managers |

---

## Stage 3: Task Graph

```
Entry: App launch | notification click | project switch

SELECT PROJECT
    ├── VIEW ENTITIES → session/:id, agent/:id, daemon/:id
    ├── SPAWN WORK → new agent → opens entity tab
    └── CONFIGURE → ports, skills

Exit: Work delegated | session closed | app quit
Fail: Port unavailable | agent crash | CMS sync fail
```

---

## Stage 4: Information Architecture

See [entity-ontology.md](file:///Users/jean-patricksmith/digital/leviathan/docs/design/entity-ontology.md) for full entity hierarchy, URI scheme, and CMS adapter interface.

---

## Stage 5: Interaction Model

**Agent Session FSM:**
```
[idle] → [planning] → [executing] → [reviewing] → [complete]
  ↑         │            │             │              │
  └─────────┴────────────┴─────────────┴──────────────┘
                    (error/cancel)
```

**Notification Routing:**
```
event → project_scope → agent_scope → panel_highlight → optional_focus_steal
```

**Project Panel:** `[collapsed] ←→ [expanded]` (click toggle)

---

## Stage 6: Component Intent

| Component | Intent | Data In | Data Out | Refresh |
|-----------|--------|---------|----------|---------|
| Project Selector | Switch full context | click | project_id | user |
| Entity Tab Bar | Navigate within project | URI | focused entity | user |
| Agent List | All agents + status | project_id | agent states | 1s poll |
| Session View | Agent conversation | session_id | messages | ws realtime |
| Daemon Panel | Processes + ports | project_id | daemon states | 5s poll |
| Workstream Board | Initiative progress | workstream_id | tasks + % | 30s poll |
| Notification Toast | Alert without focus steal | event | dismiss/focus | event |
| Spawn Wizard | Create agent/session | user input | agent_id | user |

---

## Existing Infrastructure

Dashboard Runner already exists in `community/agentping/packages/dashboard-runner/`:
- Process management with auto-restart + backoff
- Smart port selection + conflict resolution
- Health monitoring via HTTP/process checks
- Config via `dashboards.yaml`
- Manager UI at `:5175`

The dashboard spec builds ON TOP of this infrastructure.

---

## Key Insight

> "BD is a content management system, not just a task tracker. Markdown files are views based off beads. You can query them and create websites — that's where the dashboard concept comes from."

The dashboard is a **view rendered from CMS content** — same principle as docs.

---

**Status**: PROPOSAL — needs review before implementation planning.

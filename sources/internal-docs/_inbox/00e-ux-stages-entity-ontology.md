# UX Research Stages & Entity Ontology (Dashboard Spec)

> **Source:** `00-vision.md` lines 118-323 (structured tables/diagrams — most crystallized content)
> **Status:** crystallized → ready for spec promotion
> **Layer:** Skin/Services (dashboard UX, component design)
> **Interview notes:** User confirmed entity ontology is complementary to graph primitives. Ontology = project management actors; graph primitives = runtime actors. Already reflected in `lev-design` skill's 7-step pipeline.

## The 6-Stage UX Process Applied to Multi-Project Agent Dashboard

### Stage 1: Problem Framing

**Problem:** When running multiple concurrent AI agent projects, the mental model collapses. Work is split between apps (terminal, browser, IDE) with no natural grouping. Users spend more time switching than building.

**Success Criteria:**
- Identify which project a notification came from in <1s
- Zero port collisions across active projects
- Context switch between projects takes 1 click
- All agent sessions, terminals, and outputs are project-scoped
- Users report "I know where everything is"

**Who:**
- Developers running 2-5 AI agent projects simultaneously
- Power users comfortable with CLI but need visual overview
- Team leads managing swarm delegation across workstreams

### Stage 2: Jobs-to-Be-Done (JTBD)

| Job | Trigger | Context | Outcome |
|-----|---------|---------|---------|
| Know where I am | Notification sound | Deep in code, multiple tabs | Identify project + agent in <1s |
| Switch contexts cleanly | "Let me check Project B" | Working on Project A | All panels update to B's state |
| Spawn work | New feature/bug | Have an initiative/workstream | Delegate to agent with proper scope |
| Monitor progress | Periodic check | Multi-agent execution in flight | See all agent states at a glance |
| Resolve conflicts | Port collision, auth break | Dev server won't start | System prevents or auto-resolves |
| Resume session | Return from break | Lost mental model | UI shows exactly where I left off |
| Delegate portfolio | Too much in flight | PM/CEO role | Assign workstreams to sub-managers |

### Stage 3: Task Graph

Entry states: App launch, notification click, project switch
Exit states: Work delegated, session closed, app quit
Failure modes: Port unavailable, agent crash, BD sync fail

```
SELECT PROJECT
    │
    ├── VIEW ENTITIES → session/:id, agent/:id, daemon/:id
    ├── SPAWN WORK → new agent → ENTITY TAB OPENS
    └── CONFIGURE PROJECT → ports config
```

### Stage 4: Information Architecture (Entity Ontology)

**Entities (Nouns):**

| Entity | Parent | Children |
|--------|--------|----------|
| **Project** | — | Workstreams, Daemons, Skills, Config |
| **Workstream** | Project | Initiatives, Agents |
| **Initiative** | Workstream | Tasks (BD issues) |
| **Agent** | Workstream | Sessions |
| **Session** | Agent | Messages, Artifacts |
| **Daemon** | Project | Processes, Ports |
| **Skill** | Project | Rules, Templates |

**Actions (Verbs):**

| Action | Target | Result |
|--------|--------|--------|
| spawn | Agent | New session created |
| delegate | Workstream | Assigned to agent/swarm |
| pause/resume | Session | State preserved |
| configure | Project/Agent | Settings updated |
| switch | Project | All panels update |
| collapse/expand | Panel | UI state change |

**URI Scheme:**
```
agent/session:abc-123    → Session view
agent/config:worker-1    → Agent config
agent/new                → Spawn wizard
daemon/dev               → Daemon detail
project/leviathan        → Project overview
workstream/auth-refactor → Workstream board
initiative/lev-r9no      → Epic detail (BD)
```

### Stage 5: Interaction Model (FSM)

**Agent Session State:**
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

### Stage 6: Component Intent

| Role | Intent | Input | Output | Update Freq |
|------|--------|-------|--------|-------------|
| Project Selector | Switch full context | click | project_id | user-driven |
| Entity Tab Bar | Navigate within project | URI | focused entity | user-driven |
| Agent List | Show all agents + status | project_id | agent states | 1s poll |
| Session View | Show agent conversation | session_id | messages | realtime ws |
| Daemon Panel | Show processes + ports | project_id | daemon states | 5s poll |
| Workstream Board | Show initiative progress | workstream_id | tasks + % | 30s poll |
| Notification Toast | Alert without focus steal | event | dismiss/focus | event-driven |
| Spawn Wizard | Create new agent/session | user input | agent_id | user-driven |

## Key Insight (from interview)
> "BD is a content management system, not just a task tracker. Markdown files are views based off beads. You can query them and create websites — that's where the dashboard concept comes from."

## Action Items
- [ ] Promote entity ontology to `docs/design/entity-ontology.md`
- [ ] Create spec for dashboard MVP using these 6 stages
- [ ] Map entity URIs to actual BD/bead query patterns
- [ ] Align with `work` skill's entity lifecycle (ephemeral→captured→classified→crystallizing→crystallized→manifesting→completed)

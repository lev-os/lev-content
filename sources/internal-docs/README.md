# Lev Documentation

> One place to know where everything is and how it's organized.

## How to Find Things

| You want... | Go to |
|---|---|
| System architecture, module map, event model | `docs/01-architecture.md` |
| How much spec rigor does this need? (tier system) | `docs/00-process.md` |
| How to develop Lev (conventions, ownership, checklists) | `/AGENTS.md` |
| Module deep-dive (search, harness, flowmind, lifecycle, plugins) | `docs/design/` |
| Specs for systems being actively built | `docs/specs/` |
| UX architecture runs promoted for implementation | `docs/ux/` |
| Working proposals and ephemeral agent artifacts | `.lev/pm/` |
| What a specific crate does (spec, parity, changelog) | `crates/<crate>/docs/` |
| Working drafts, research, voice transcripts | `docs/_inbox/` |
| Historical reference | `docs/_archive/` (read-only) |

---

## Documentation Layers

There are three kinds of docs and they live in different places.

### 1. System-wide docs (`docs/`)

Cross-cutting architecture, design decisions, and how modules work **together**.

```
docs/
├── README.md                     # this file — the index
├── 00-process.md                 # dev process, tier system, shovel-ready criteria
├── 01-architecture.md            # canonical system architecture (C1-C4, vernacular, contracts)
│
├── design/                       # system design — locked decisions, extracted references
│   ├── vernacular.md             # LOCKED: 5 core + 7 reactive terms, retired terms
│   ├── contracts.md              # LOCKED: 7 standards, config chain, plugins, bridge pattern
│   ├── cli-surface.md            # LOCKED: canonical CLI surface + compatibility policy
│   ├── graph-primitives.md       # LOCKED: View, Claim, Evidence, TruthState, Gate, Tick, Multi-Axis Depth
│   ├── dimension-model.md        # PROPOSAL: N-dimensional entity coordinates, append-only graph, projections
│   ├── core-index.md             # search: LEANN + ck + mgrep
│   ├── core-poly.md              # poly binder + daemon/CLI/MCP/SDK binding model
│   ├── core-sdk.md               # harness, providers, platform bridge
│   ├── core-flowmind.md          # FlowMind compiler, triggers, kernel
│   ├── core-lifecycle.md         # events bus, LevEvent, FSMs, graph ops
│   ├── core-other.md             # plugins, config, domain, daemons
│   └── actor-graph.md            # actor-graph architecture proposal
│
├── specs/                        # staged specs — systems actively being built
│   ├── spec-lev-kernel.md             # 9-primitive enforcement kernel
│   ├── spec-entity-lifecycle.md       # entity lifecycle system
│   ├── spec-router-flowmind-parser.md # real-time intent detection
│   ├── spec-platform-integration.md   # adapter registry, hooks, deploy plugin boundary (`plugins/deploy`)
│   ├── spec-self-learning-pipeline.md # 7-phase learning pipeline
│   └── ... (15 total)                 # see docs/specs/ for full list
│
├── ux/                           # promoted UX runs ready for implementation handoff
│   └── lev-workspace-20260212/   # chat/voice/control-plane expanded -> MVP-reduced UX package
│
├── _inbox/                       # working inputs, not canonical design
│   ├── 00-vision.md              # raw voice transcripts
│   └── lev-ui/                   # UX spec drafts
│
└── _archive/                     # historical reference (read-only)
    ├── 01-kernel-core.md         # archived PRD stubs
    ├── 02-marketing.md           # archived marketing brief
    └── ...
```

### 2. Crate-local docs (`crates/<name>/docs/`)

Self-contained specs, changelogs, and manuals for a specific crate or crate group. Lives with the code because these are often git submodules.

```
crates/lev-agentfs/
├── README.md                     # crate root (Cargo/GitHub convention)
└── docs/
    ├── SPEC.md                   # SQLite schema spec (upstream)
    ├── LEVFS.md                  # Lev integration spec
    ├── FEATURE_PARITY.md         # quantitative implementation tracking
    ├── MANUAL.md                 # CLI reference
    ├── CHANGELOG.md              # version history
    └── TESTING.md                # test guide
```

### 3. Agent working area (`.lev/pm/`)

Ephemeral workspace for agents. Specs, proposals, handoffs, and reports live here until promoted.

```
.lev/pm/
├── specs/                        # in-flight specs (draft → review → approved)
├── proposals/                    # pre-spec decision requests
├── handoffs/                     # session continuity artifacts
├── reports/                      # research and analysis outputs
└── README.md                     # full taxonomy
```

### Promotion Workflow

```
.lev/pm/specs/spec-foo.md          ← agent drafts/reconciles spec
        │
        │ decision to promote as canonical design
        ▼
docs/specs/spec-foo.md             ← canonical design spec
        │
        │ implementation evolves
        ▼
docs/specs/spec-foo.md             ← updated in same PR as code changes
```

### Relationship between layers

```
docs/design/core-flowmind.md      ← "how FlowMind works, locked decisions"
                │
                │ points at
                ▼
crates/lev-agentfs/docs/SPEC.md   ← "what agentfs IS right now"
.lev/pm/*                         ← "ephemeral planning and reconciliation artifacts"
```

---

## Crate Documentation Index

| Crate | Docs | Description |
|---|---|---|
| `lev-agentfs` | `crates/lev-agentfs/docs/` | SQLite-backed FUSE/NFS filesystem for agent sandboxing |
| `lev-reactive` | `crates/lev-reactive/` | Sync/async hook registry, plugin system |
| `lev-tui-core` | `crates/lev-tui-core/` | TUI foundation (ratatui) |
| `lev-tui-theme` | `crates/lev-tui-theme/` | Terminal theming and color system |
| `lev-tui-widgets` | `crates/lev-tui-widgets/` | Custom TUI widget library |
| `lev-tui` | `crates/lev-tui/` | TUI orchestrator |
| `lev-desktop` | `crates/lev-desktop/` | Desktop application |
| `lev-ui-universal` | `crates/lev-ui-universal/` | Cross-platform UI abstractions |

---

## Interpretation Rules

- `docs/design/*` = locked design decisions and extracted architecture references. Changes slowly.
- `docs/specs/*` = canonical design specs for systems being actively built.
- `docs/_inbox/*` = working inputs, research drafts, content to mine. Not canonical design.
- `docs/_archive/*` = historical reference only. Read-only.
- `.lev/pm/` = ephemeral PM workspace for planning, proposals, handoffs, reports, and reconciliation traces.
- `/AGENTS.md` = operational instructions for agents developing Lev.
- Crate `README.md` = entry point. Crate `docs/` = everything else.

---

## Refinement Passes

| Pass | Lens | Status |
|------|------|--------|
| **L1: Layout Coherence** | Names match jobs, modules in right places, dead code removed | IN PROGRESS |
| **L2: Contract Compliance** | Every module follows Core System Contract (SRP, standalone, config.yaml) | NOT STARTED |
| **L3: Integration Validation** | Validation gates, TDD critical path, end-to-end smoke tests | NOT STARTED |

---

## Key Decisions

### Session 7 (2026-02-04)

- **Graph Primitives LOCKED**: View, Claim, Evidence, TruthState, Proposal
- **L0-L3 Depth Model** — Domain-agnostic depth abstraction
- **Canonical design lives in `~/lev/docs`** — graph primitives describe target runtime semantics
- **Ephemeral is a STATE** — Not an entity type; entities transition through states
- **Graph Operations** — Every change is a patch (add_entity, patch_entity, add_claim, add_link)
- **Gate Taxonomy** — Scope, Safety, Feasibility, Standards, Progress
- **Tick Loop** — Ingest > Observe > Propose > Gate > Apply > Update > Emit

### Session 6 (2026-02-03)

- **observe/ > events/** — Canonical event bus renamed
- **Event Architecture LOCKED**: events/ (spine) + triggers/ (execution) + flowmind/ (declaration) + lifecycle/ (state)
- **Single LevEvent schema** — Replaces LifecycleEvent, Event<T>, CloudEvents
- **Kernel = ValidationGateExecutor** — Lives in events/, ~300 LOC gap

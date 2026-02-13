---
module: "lifecycle"
design_refs:
  - "docs/design/core-lifecycle.md"
  - "docs/design/vernacular.md"
code_refs:
  - "src/core/lifecycle/**"
validation_gates:
  - "gate:lifecycle-fsm-completeness"
---
id: spec-entity-lifecycle
title: Entity Lifecycle System
status: draft
created: 2026-02-01
updated: 2026-02-11
group: entity-lifecycle
sources:
  - spec-entity-lifecycle-system.md (base)
  - spec-entity-dashboard-consolidation.md
supersedes:
  - spec-entity-lifecycle-system.md
  - spec-entity-dashboard-consolidation.md
---

# Spec: Entity Lifecycle System

**Type:** Meta-Spec (defines the spec system itself)
**Owner:** Lev Core

---

## 1. Business Case

### Problem Statement

The Lev ecosystem lacks a canonical definition of what a "spec" is, when to use one, and how entities flow through their lifecycle. Current issues:

1. **No spec for specs** - No meta-definition of the specification system
2. **Ephemeral treated as entity** - Should be a STATE, not an entity type
3. **Command fragmentation** - `/plan`, `/planbd`, `/planfile`, `/agent-plan` all do planning
4. **No clear entity→command mapping** - Hard to know which slash command tracks which entity
5. **Event naming inconsistency** - IAbstractHooks events don't align with lifecycle stages

### User Value

1. **Clarity** - Know when a feature needs a spec vs when it doesn't
2. **Consistency** - One command per entity type
3. **Traceability** - Every entity has a corresponding slash command
4. **Auditability** - Clear mapping of events to lifecycle transitions

### Strategic Alignment

- **Idea-to-Spec Lifecycle:** Formalizes the exploration → commitment → execution flow
- **tech-leads-club patterns:** Adopts WHEN/THEN/SHALL format, P1/P2/P3 priorities
- **PATCH > ADD:** Extends existing /work skill, doesn't create new entry points

---

## 2. Entity Definition

### What is an Entity?

An **entity** is any trackable unit of work in the Lev ecosystem:

| Entity Type  | Description                 | Lifecycle Applies? | Spec Required?      |
| ------------ | --------------------------- | ------------------ | ------------------- |
| **Idea**     | Exploration artifact        | Yes                | No (might die)      |
| **Research** | Structured investigation    | Yes                | No (captured stage) |
| **Proposal** | Decision request            | Yes                | No (crystallizing)  |
| **Spec**     | Feature definition          | Yes                | N/A (IS the spec)   |
| **Design**   | Implementation architecture | Yes                | If complex          |
| **Epic**     | BD execution tracker        | Yes                | Usually             |
| **Task**     | Atomic work unit            | Partial            | No                  |
| **Session**  | Agent conversation          | Partial            | No                  |

### Ephemeral is a STATE, Not an Entity

**CRITICAL DESIGN DECISION:**

```
WRONG:  entity_type = "ephemeral" | "idea" | "spec" | ...
RIGHT:  entity_state = "ephemeral" | "captured" | "crystallizing" | ...
```

**Ephemeral means:** No persistent artifact yet, brainstorming mode, can be discarded.

**Transition:** When an ephemeral conversation produces something worth keeping, it CAPTURES to an idea/report.

---

## 3. Lifecycle States

### Canonical State Machine

```
┌──────────┐    ┌──────────┐    ┌─────────────┐    ┌─────────────┐    ┌────────────┐    ┌───────────┐
│EPHEMERAL │ →  │ CAPTURED │ →  │CRYSTALLIZING│ →  │ CRYSTALLIZED│ →  │ MANIFESTING│ →  │ COMPLETED │
└──────────┘    └──────────┘    └─────────────┘    └─────────────┘    └────────────┘    └───────────┘
     │               │                │                   │                 │                 │
   [none]        [report]        [proposal]           [spec]            [epic]          [archive]
                                                      [design]          [tasks]
```

> **⚠️ CONFLICT — Two Lifecycle Models (needs review)**
>
> This spec defines the **entity lifecycle** above (ephemeral → captured → crystallizing → crystallized → manifesting → completed). The superseded `spec-entity-dashboard-consolidation` defines a separate **spec document lifecycle** for the BD type system: `draft → review → approved → implementing`.
>
> These may be **COMPLEMENTARY** (entity lifecycle = what state the _work_ is in; spec lifecycle = what state the _document_ is in). A spec document in `draft` status tracks work that is in the `crystallizing` entity state; an `approved` spec tracks work transitioning to `manifesting`.
>
> **Resolution needed:** Confirm whether both models coexist (entity lifecycle + document lifecycle) or unify into one. If coexisting, define the mapping between them explicitly.
>
> See also: Section 11.7 (Spec Lifecycle BD Type) for the full document lifecycle definition from the dashboard consolidation spec.

### State Definitions

| State             | Description                  | Entry Condition     | Exit Condition        | Artifact               |
| ----------------- | ---------------------------- | ------------------- | --------------------- | ---------------------- |
| **ephemeral**     | No commitment, brainstorming | Conversation starts | Decision to capture   | None                   |
| **captured**      | Information gathered         | Worth preserving    | Ready for design      | `report.md`            |
| **crystallizing** | Designing solution           | Report complete     | Design approved       | `proposal.md`          |
| **crystallized**  | Committed to build           | Proposal approved   | Implementation starts | `spec.md`, `design.md` |
| **manifesting**   | Active development           | Spec exists         | All tasks done        | `epic`, `tasks`        |
| **completed**     | Work finished                | Tasks closed        | Archived              | Archive                |

### State Transitions

```typescript
type LifecycleState = 'ephemeral' | 'captured' | 'crystallizing' | 'crystallized' | 'manifesting' | 'completed'

interface StateTransition {
  from: LifecycleState
  to: LifecycleState
  trigger: string
  guard?: string // Condition that must be true
  action?: string // Side effect to execute
}

const TRANSITIONS: StateTransition[] = [
  { from: 'ephemeral', to: 'captured', trigger: 'capture', action: 'create_report' },
  { from: 'captured', to: 'crystallizing', trigger: 'design', guard: 'report_exists' },
  { from: 'crystallizing', to: 'crystallized', trigger: 'approve', guard: 'proposal_approved' },
  { from: 'crystallized', to: 'manifesting', trigger: 'implement', guard: 'spec_exists', action: 'create_epic' },
  { from: 'manifesting', to: 'completed', trigger: 'complete', guard: 'all_tasks_done', action: 'archive' },

  // Backward transitions (rework)
  { from: 'crystallizing', to: 'captured', trigger: 'rework', action: 'update_report' },
  { from: 'crystallized', to: 'crystallizing', trigger: 'redesign', action: 'update_proposal' },
  { from: 'manifesting', to: 'crystallized', trigger: 'replan', action: 'update_spec' },
]
```

---

## 4. Command Hierarchy

### Design Principle: /work as Main, Others as Subcommands

```
/work                    # Main entry point - auto-routes by lifecycle stage
├── /work research       # Force captured stage (or use /research)
├── /work design         # Force crystallizing stage
├── /work plan           # Force crystallized stage (or use /plan)
└── /work handoff        # Force manifesting stage (or use /handoff)

/interview               # Wizard-mode questioning (input to any stage)
/research                # Shortcut to /work research
/plan                    # Shortcut to /work plan
/handoff                 # Shortcut to /work handoff
```

### Entity → Command Mapping

| Entity State  | Primary Command       | Shortcut             | Artifact Created       |
| ------------- | --------------------- | -------------------- | ---------------------- |
| ephemeral     | (inline conversation) | -                    | None                   |
| captured      | `/work research`      | `/research`          | `report.md`            |
| crystallizing | `/work design`        | `/interview` (input) | `proposal.md`          |
| crystallized  | `/work plan`          | `/plan`              | `spec.md`, `design.md` |
| manifesting   | `/work execute`       | `/bd`                | Epic, Tasks            |
| completed     | `/work archive`       | `/handoff`           | Archive                |

### Command Consolidation

**DELETE these redundant commands:**

```
~/.claude/commands/plan.md        # → use /work plan
~/.claude/commands/planbd.md      # → use /work plan --bd
~/.claude/commands/planfile.md    # → use /work plan --file
~/.claude/commands/agent-plan.md  # → use /work plan --team
```

**KEEP as shortcuts:**

```
~/.claude/commands/research.md    # Shim to /work research
~/.claude/commands/interview.md   # Wizard questioning skill
~/.claude/commands/handoff.md     # Shim to /work handoff
```

**UPDATE:**

```
~/.claude/commands/lev/plan.md    # Point to /work plan
```

---

## 5. Spec Definition

### When Does a Feature Need a Spec?

**DECISION TREE:**

```
Is it a bug fix or small tweak?
├─ YES → No spec needed (just BD task)
└─ NO → Continue...

Does it affect >3 files?
├─ NO → No spec needed (inline implementation)
└─ YES → Continue...

Does it introduce new architecture/patterns?
├─ YES → SPEC REQUIRED
└─ NO → Continue...

Is there uncertainty about approach?
├─ YES → SPEC REQUIRED (crystallize thinking)
└─ NO → Continue...

Will multiple agents/sessions work on it?
├─ YES → SPEC REQUIRED (canonical reference)
└─ NO → Spec optional (design.md may suffice)
```

### Spec Required Indicators

| Indicator                            | Spec Required | Rationale                     |
| ------------------------------------ | ------------- | ----------------------------- |
| New feature with multiple components | Yes           | Canonical reference needed    |
| Architectural change                 | Yes           | Decisions need documentation  |
| Cross-cutting concern                | Yes           | Multiple touchpoints          |
| Complex business logic               | Yes           | Requirements must be explicit |
| Multi-session work                   | Yes           | Continuity across agents      |
| Simple CRUD endpoint                 | No            | Self-evident from code        |
| Bug fix                              | No            | Issue tracker sufficient      |
| Config change                        | No            | Inline documentation          |
| Refactor without behavior change     | No            | Code is the spec              |

### Spec Template (Hybrid: tech-leads-club + Lev)

```markdown
# Spec: {Feature Name}

**ID:** spec-{slug}
**Status:** Draft | Review | Approved | Implementing | Implemented
**Owner:** {team/person}
**Created:** {date}
**Updated:** {date}

---

## 1. Business Case

### Problem Statement

[2-3 sentences: What pain point? Why now?]

### User Value

[Measurable outcomes]

### Strategic Alignment

[How fits roadmap]

---

## 2. User Stories

### P1: {Story Title} ⭐ MVP

**User Story:** As a [role], I want [capability] so that [benefit].

**Why P1:** [Why critical for MVP]

**Acceptance Criteria:**

1. WHEN [action] THEN system SHALL [behavior]
2. WHEN [action] THEN system SHALL [behavior]
3. WHEN [edge case] THEN system SHALL [graceful handling]

**Independent Test:** [How to verify this story works alone]

---

### P2: {Story Title}

**User Story:** As a [role], I want [capability] so that [benefit].

**Acceptance Criteria:**

1. WHEN [action] THEN system SHALL [behavior]

---

## 3. Architecture Overview

### System Context

[How fits existing architecture - diagram if helpful]

### Component Design

[Major components and interactions]

### Technology Choices

[Tech decisions with rationale]

---

## 4. Task Breakdown

### T1: {Create X Interface}

**What:** [One sentence, one deliverable]
**Where:** `src/path/to/file.ts`
**Depends:** None
**Reuses:** `src/existing/BaseInterface.ts`
**Tools:** MCP: filesystem | Skill: none
**Done when:**

- [ ] Interface defined
- [ ] Types exported
- [ ] No TypeScript errors

---

### T2: {Implement Y Service} [P]

**What:** [One deliverable]
**Where:** `src/services/YService.ts`
**Depends:** T1
**Reuses:** `src/services/BaseService.ts`
**Tools:** MCP: filesystem | Skill: api-design
**Done when:**

- [ ] Implements interface from T1
- [ ] Handles error cases
- [ ] Unit test passes

---

## 5. Validation Gates

### Acceptance Criteria (DoR)

yaml
ENTRY:

- [Prerequisites]

DONE_STATE:

- [Files that exist]
- [Artifacts produced]

CRITERIA:

- [Validation steps]

TOUCHPOINTS:

- [Related systems]

INTEGRATION:

- [How fits Lev patterns]

E2E:

- [Verification command]

### Success Metrics

- [Quantitative measure]
- [Quantitative measure]

---

## 6. Implementation Tracking

**BD Epic:** {epic-id}
**Related Specs:** [links]
**Related Designs:** [links]
```

---

## 6. Event Naming Audit

### Current IAbstractHooks Events (15)

| Event              | Category   | Lifecycle Stage          | Rename? |
| ------------------ | ---------- | ------------------------ | ------- |
| session_start      | Session    | ephemeral                | No      |
| session_checkpoint | Session    | any                      | No      |
| session_handoff    | Session    | manifesting→completed    | No      |
| session_end        | Session    | any                      | No      |
| pre_tool_use       | Tool       | any                      | No      |
| post_tool_use      | Tool       | any                      | No      |
| tool_error         | Tool       | any                      | No      |
| task_start         | Task       | manifesting              | No      |
| task_progress      | Task       | manifesting              | No      |
| task_complete      | Task       | manifesting              | No      |
| task_blocked       | Task       | manifesting              | No      |
| dor_check          | Validation | crystallized→manifesting | No      |
| validation_pass    | Validation | any transition           | No      |
| validation_fail    | Validation | any transition           | No      |
| escalation         | Validation | any                      | No      |

### Phase 1 Events (New) - Naming Audit

| Proposed Event   | Lifecycle Stage | Final Name         | Notes               |
| ---------------- | --------------- | ------------------ | ------------------- |
| prompt_submit    | any             | `prompt_submit`    | Matches Claude Code |
| prompt_rendered  | any             | `prompt_rendered`  | After template      |
| prompt_complete  | any             | `prompt_complete`  | After LLM response  |
| subagent_spawn   | manifesting     | `subagent_spawn`   | Before spawn        |
| subagent_start   | manifesting     | `subagent_start`   | Execution begins    |
| subagent_stop    | manifesting     | `subagent_stop`    | Execution ends      |
| subagent_message | manifesting     | `subagent_message` | Inter-agent         |
| pre_compact      | any             | `pre_compact`      | Matches Claude Code |
| post_compact     | any             | `post_compact`     | After compaction    |
| context_overflow | any             | `context_overflow` | Approaching limit   |

### Lifecycle Transition Events (Proposal)

| Event                   | Trigger                    | Guard          |
| ----------------------- | -------------------------- | -------------- |
| `lifecycle_capture`     | ephemeral→captured         | report created |
| `lifecycle_crystallize` | captured→crystallizing     | design started |
| `lifecycle_commit`      | crystallizing→crystallized | spec approved  |
| `lifecycle_manifest`    | crystallized→manifesting   | epic created   |
| `lifecycle_complete`    | manifesting→completed      | all tasks done |

---

## 7. Leviathan Skills Audit

### Skills to Audit Post-Spec

After this spec is approved, audit these skills for alignment:

| Skill           | Audit For               | Action                       |
| --------------- | ----------------------- | ---------------------------- |
| `lev-lifecycle` | State machine alignment | Update to match spec         |
| `lev-builder`   | PATCH > ADD pattern     | Verify consistent            |
| `work`          | Command hierarchy       | Ensure /work is entry point  |
| `planning`      | DoR enforcement         | Verify 6 sections            |
| `lev-cdo`       | Graph operations        | Add lifecycle events         |
| `lev-research`  | Captured stage          | Ensure creates report.md     |
| `interview`     | Input collection        | Verify works with all stages |
| `lev-align`     | Drift detection         | Add lifecycle drift          |

### Skills Consolidation

| Keep           | Merge Into | Delete                       |
| -------------- | ---------- | ---------------------------- |
| `work`         | -          | -                            |
| `lev-research` | -          | -                            |
| `interview`    | -          | -                            |
| -              | `work`     | `planning` (if redundant)    |
| -              | -          | `lev-examples` (no SKILL.md) |

### Skill Audit Results (from dashboard consolidation)

**17 KEEP / 1 DELETE**

DELETE:

- `lev-examples` (no SKILL.md, minimal value)

Phase 2 consolidation:

- `lev-clwd` conversation search → `lev-find` unified search

### Command Consolidation

| Keep         | Redirect To      | Delete                                |
| ------------ | ---------------- | ------------------------------------- |
| `/work`      | -                | -                                     |
| `/research`  | `/work research` | -                                     |
| `/interview` | -                | -                                     |
| `/plan`      | `/work plan`     | -                                     |
| `/handoff`   | `/work handoff`  | -                                     |
| -            | `/work plan`     | `/planbd`, `/planfile`, `/agent-plan` |

---

## 8. Requirements

### Functional Requirements

**MUST:**

- [ ] Define spec template with WHEN/THEN/SHALL format
- [ ] Define entity lifecycle state machine
- [ ] Map each state to slash command
- [ ] Define when spec is required
- [ ] Audit and rename events for consistency
- [ ] Consolidate redundant commands

**SHOULD:**

- [ ] Add lifecycle transition events to IAbstractHooks
- [ ] Create spec validation gate (before manifesting)
- [ ] Update lev-lifecycle skill to match

**MAY:**

- [ ] Auto-generate spec from interview output
- [ ] Spec compliance scoring
- [ ] Spec freshness tracking

### Non-Functional Requirements

| Requirement       | Target                                      |
| ----------------- | ------------------------------------------- |
| Learning curve    | <10 min to understand when spec needed      |
| Command discovery | One entry point (/work)                     |
| Backwards compat  | Existing commands still work (as redirects) |

---

## 9. Validation Gates

### Acceptance Criteria

- [ ] Spec template defined with all sections
- [ ] Lifecycle states documented with transitions
- [ ] Command hierarchy documented
- [ ] Event names audited and finalized
- [ ] Skills audit checklist created
- [ ] Decision tree for "when spec needed" documented

### Success Metrics

- **Clarity:** New contributors understand spec system in <10 min
- **Adoption:** 100% of new features >3 files use spec
- **Consistency:** All lifecycle transitions logged as events

---

## 10. Implementation Tracking

### Task Breakdown

| ID  | Task                                   | Priority | Depends |
| --- | -------------------------------------- | -------- | ------- |
| T1  | Finalize spec template                 | P0       | -       |
| T2  | Update lev-lifecycle skill             | P0       | T1      |
| T3  | Add lifecycle events to IAbstractHooks | P1       | T1      |
| T4  | Consolidate plan commands              | P1       | T1      |
| T5  | Update work skill routing              | P1       | T4      |
| T6  | Skills audit (8 skills)                | P2       | T1      |
| T7  | Documentation update                   | P2       | T1-T6   |

### Related Artifacts

- **Idea-to-Spec Lifecycle:** `~/.lev/pm/proposals/20260129-idea-to-spec-lifecycle.md`
- **Entity Dashboard Spec:** `~/.lev/pm/specs/entity-dashboard-consolidation-spec.md`
- **tech-leads-club Reference:** `/tmp/agent-skills/skills/(development)/spec-driven-dev/`
- **Work Skill:** `~/.claude/skills/work/SKILL.md`

---

## Appendix A: Decision Tree Flowchart

```
                    ┌─────────────────────────────┐
                    │ New work request received   │
                    └─────────────────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────────┐
                    │ Is it a bug fix or tweak?   │
                    └─────────────────────────────┘
                         │ YES              │ NO
                         ▼                  ▼
              ┌──────────────────┐  ┌─────────────────────────┐
              │ BD task only     │  │ Does it affect >3 files?│
              │ No spec needed   │  └─────────────────────────┘
              └──────────────────┘       │ NO              │ YES
                                         ▼                  ▼
                              ┌──────────────────┐  ┌─────────────────────────┐
                              │ Inline impl      │  │ New architecture?       │
                              │ No spec needed   │  └─────────────────────────┘
                              └──────────────────┘       │ NO              │ YES
                                                         ▼                  ▼
                                              ┌──────────────────┐  ┌──────────────────┐
                                              │ Uncertainty?     │  │ ⭐ SPEC REQUIRED │
                                              └──────────────────┘  └──────────────────┘
                                                   │ NO              │ YES
                                                   ▼                  ▼
                                        ┌──────────────────┐  ┌──────────────────┐
                                        │ Multi-session?   │  │ ⭐ SPEC REQUIRED │
                                        └──────────────────┘  └──────────────────┘
                                             │ NO              │ YES
                                             ▼                  ▼
                                  ┌──────────────────┐  ┌──────────────────┐
                                  │ design.md only   │  │ ⭐ SPEC REQUIRED │
                                  │ Spec optional    │  └──────────────────┘
                                  └──────────────────┘
```

---

## 11. Dashboard & UX Consolidation

> Consolidated from `spec-entity-dashboard-consolidation.md`. Contains interview decisions, dashboard UX patterns, and operational details for the /work consolidation.

### 11.1 Graph Diff UX

**Decision:** Inline ASCII tree with mode switching

```
[r] relevance  [t] by-type  [d] dependency-order

├── clawd-v1h2.5 (BD) open → closed
│   └── core/graph/dag.ts (move to harness/)
├── clawd-v1h2.6 (BD) open → closed
└── docs/layout.md (update imports)

[r/t/d] switch | [y] apply | [n] cancel
```

Agent chooses mode at runtime. User can switch with shortcuts.

### 11.2 Wizard Mode Integration

**Decision:** Update /guideme + /think, auto-route by complexity

- Simple → inline response
- Complex → skill://think (thinking-parliament)
- Specs are markdown with inline flowmind/yaml
- No new FSM infrastructure

### 11.3 Validation Gates as Pipeline

**Decision:** Pipeline with numbered workflow steps

```
<skill>/workflows/##-{slug}.md

01-intake.md
02-wizard.md (interview)
03-commit.md
04-validation.md (prompt-based gate)
05-epic-creation.md
```

Validation gate is a PROMPT before creating epics.

### 11.4 Exec Submenu (e)

**Decision:** Show menu when no target

```
e → show: lev | ralph | cs | team
e lev <epic> → lev exec --epic <epic>
e ralph <epic> → ralph-loop.sh --epic <epic>
e cs "<prompt>" → csx "<prompt>"
e team "<prompt>" → cs team mode
```

### 11.5 Dashboard Footer

**Decision:** Every significant turn (after tool use or major response)

```
───────────────────────────────────────────────────
🎯 conf: 0.XX | 📦 N entities | ⚡ stage
├── id: title [stage]
└── session: XX% context
───────────────────────────────────────────────────
```

### 11.6 Plugin Discovery

**Decision:** ls + cache

```bash
ls ~/lev/plugins/ + ~/.claude/skills/*execution*
→ Cache: ~/.claude/lev-plugins.yaml (1-day mtime refresh)
```

### 11.7 Spec Lifecycle (BD Type)

> **⚠️ CONFLICT** — See Section 3 conflict note. This defines a _document-level_ lifecycle separate from the entity lifecycle state machine.

**Decision:** draft → review → approved → implementing

```yaml
# ~/lev/.beads/types/spec.yaml
name: spec
extends: entity
description: Persistent specification with validation gates
bd_type: epic
labels: [spec, blueprint]

lifecycle:
  states: [draft, review, approved, implementing]
  default: draft
  transitions:
    draft:
      - to: review
        trigger: submit_for_review
    review:
      - to: approved
        trigger: approve
        actions: [create_validation_gate]
      - to: draft
        trigger: request_changes
    approved:
      - to: implementing
        trigger: start_implementation
        actions: [create_epics_from_spec]
```

### 11.8 System Prompt Location

**Decision:** Update all three:

- ~/.claude/CLAUDE.md
- ~/.claude-sneakpeek/claudesp/config/CLAUDE.md
- ~/clawd/AGENTS.md

Prune/synthesize existing content.

### 11.9 Guideme → Interview Rename

**Decision:**

- Rename `/guideme` → `/interview`
- Create `skills/interview` (shim pattern)
- Do NOT merge with thinking-parliament

---

## 12. Dashboard Shortcuts

| Key   | Skill                     | Description                      |
| ----- | ------------------------- | -------------------------------- |
| (s)   | skill://lev-find          | Search (all scopes)              |
| (p)   | skill://work/proposal     | Proposal → BD                    |
| (t)   | skill://think             | Think (multi-agent deliberation) |
| (u)   | skill://work/update       | Update (graph diff preview)      |
| (e)   | skill://agentic-execution | Execute (submenu)                |
| (o)   | skill://oracle            | Multi-model routing              |
| (art) | prior art check           | Hidden, still works              |

**Removed:** (c) crystallize - handled by lifecycle stages

---

## 13. BD as CMS Pattern

### Source of Truth

```yaml
handlers:
  default: bd:// # BD is source of truth
  mirrors: [file://...] # Files are views (generated)
```

### Template Rendering

```yaml
file_template: |
  id: "{slug}"
  title: "{title}"
  status: "{lifecycle_stage}"
  ...
```

### Hooks

```yaml
hooks:
  post_create:
    - mirror_to_filesystem # Auto-generate markdown
    - index_for_search
  on_stage_change:
    - validate_transition
    - update_trajectory
```

---

## 14. Pipeline Workflow Pattern

Each step in `<skill>/workflows/##-{slug}.md`:

```
01-intake.md      → Initial request capture
02-wizard.md      → Interview (JTBD, first principles, etc.)
03-commit.md      → Commit to BD, generate files
04-validation.md  → Validation gate prompt
05-epic-creation.md → Create BD epics from spec
```

---

## 15. Files to Create / Delete / Update

### Files to Create

1. **~/.local/bin/csx** — Claude Sneakpeek headless executor

```bash
#!/bin/bash
# Prepends skill://agentic-execution to prompts
claude --skill agentic-execution "$@"
```

2. **~/.claude/lev-plugins.yaml**

```yaml
plugins:
  manifestation:
    - name: sdlc
      entry: skill://agentic-execution
    - name: research
      entry: skill://lev-research
    - name: docs
      entry: skill://lev-intake
    - name: skill
      entry: skill://skill-builder

  execution:
    - name: lev
      entry: lev exec
    - name: ralph
      entry: ralph-loop.sh
    - name: cs
      entry: csx
    - name: team
      entry: skill://claudesp-internals/team

last_refresh: 2026-02-01T00:00:00Z
```

3. **~/.claude/skills/interview/SKILL.md** — New skill (shim from /interview command)

4. **~/.claude/commands/interview.md** — Rename from guideme.md

### Files to Delete

```bash
~/.claude/commands/plan.md
~/.claude/commands/planbd.md
~/.claude/commands/planfile.md
~/.claude/commands/agent-plan.md
~/.claude/skills/plan/
~/.claude/skills/planning/
~/.claude/skills/lev-examples/  # No SKILL.md, low value
```

### Files to Update

- **~/.claude/CLAUDE.md** — Add dashboard shortcuts section, footer format, shortcut→skill:// mapping
- **~/.claude-sneakpeek/claudesp/config/CLAUDE.md** — Mirror updates from ~/.claude/CLAUDE.md
- **~/clawd/AGENTS.md** — Prune/synthesize existing content, add dashboard shortcuts, update BD discipline section
- **~/.claude/skills/work/SKILL.md** — Add shortcut parser, pipeline workflow references, BD-native flow
- **~/.claude/commands/guideme.md** — Rename to interview.md, update to shim pattern

---

## 16. Claudesp Reconciliation

**Skills:** 100% symlinked (optimal, no changes needed)

**Commands:** 20 files are identical copies → convert to symlinks

- Exception: `sync-skills.md` (keep as file, claudesp-specific)

**lev/ subdirectory:** Only in ~/.claude (intentional separation)

---

## 17. Execution Order

```
Phase 1: Foundation
├── [ ] Create ~/.local/bin/csx
├── [ ] Create ~/.claude/lev-plugins.yaml
├── [ ] Create ~/lev/.beads/types/spec.yaml

Phase 2: Rename/Delete
├── [ ] Rename guideme.md → interview.md
├── [ ] Create skills/interview/SKILL.md
├── [ ] Delete plan*, planbd, planfile, agent-plan
├── [ ] Delete skills/plan, skills/planning
├── [ ] Delete skills/lev-examples

Phase 3: Updates
├── [ ] Update ~/.claude/CLAUDE.md (shortcuts + footer)
├── [ ] Update claudesp CLAUDE.md (mirror)
├── [ ] Update ~/clawd/AGENTS.md (prune + add)
├── [ ] Update work/SKILL.md (pipeline pattern)

Phase 4: Claudesp Sync
├── [ ] Convert 20 command copies to symlinks
```

---

## Open Items

1. Verify ralph-loop.sh streaming mode works
2. Test csx headless executor
3. Finalize interview skill content
4. **Resolve lifecycle state conflict** (Section 3 / Section 11.7)

---

**Ready for:** User review — lifecycle conflict resolution required before implementation

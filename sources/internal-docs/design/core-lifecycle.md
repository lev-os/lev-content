# 07 - Core: Events, Lifecycle & State

**Status**: ✅ LOCKED (Session 6)
**Epic**: lev-cpnv (Lifecycle FSM Consolidation)
**Last Updated**: 2026-02-04

---

## Scope

**What this covers:**
- Event bus and persistence (renamed from observe/)
- Entity lifecycle FSMs
- Validation gates execution
- State projections

**Modules:**
```
~/lev/core/
├── events/             # Canonical event bus (RENAMED from observe/)
│   ├── src/
│   │   ├── bus.ts              # EventBus (singleton, glob patterns)
│   │   ├── log.ts              # EventLog (JSONL persistence)
│   │   ├── schema.ts           # LevEvent (ONE schema)
│   │   ├── cursors/            # Durable views (queue, session)
│   │   └── bridges/            # Footer bridge + event adapters
│   └── ValidationGateExecutor.ts  # ~300 LOC NEEDED
│
├── lifecycle/          # FSMs + state projection
│   ├── src/
│   │   ├── machines/           # XState FSMs
│   │   │   ├── execution.ts    # ✅ COMPLETE
│   │   │   └── entity.ts       # ⏳ STUB
│   │   ├── gates/              # Validation gate definitions
│   │   ├── hooks/              # Lifecycle hooks
│   │   └── projections/        # State derived from events
│   └── queue.ts                # Durable queue
│
└── triggers/           # Rule engine — see design/core-flowmind.md
```

---

## CORE PRINCIPLE (LOCKED)

**observe/ → events/** — "Just call it events, that's what it is."

Fresh-eyes test results:
- Engineers searching "event system" → confusion 3/10 with observe/
- Engineers searching "lifecycle" → found lifecycle/ easily (2/10)
- Rename improves discoverability

---

## Event Architecture (LOCKED)

### Single Event Schema: LevEvent

```typescript
interface LevEvent<T = unknown> {
  version: number;               // 1
  id: string;                    // ULID
  type: string;                  // "session.died", "cron.tick"
  source: string;                // "provider:git", "flowmind:hook"
  time: string;                  // ISO8601
  data: T;
}
```

**Replaces**: LifecycleEvent (observe), Event<T> (types), CloudEvents emit coupling (agent-adapter)

### Single Event Log

**Location**: `$XDG_DATA_HOME/lev/events.jsonl`

**Consolidates**:
- `.lev/events.jsonl` (legacy read compatibility)
- `.lev/agent-events.jsonl` (legacy read compatibility)
- `$XDG_CONFIG_HOME/lev/events.jsonl` (legacy read compatibility)
- `$XDG_STATE_HOME/lev/events.jsonl` (legacy read compatibility)
- `$XDG_DATA_HOME/lev/events.jsonl` (canonical write path)

**Cursors for views**:
- Queue cursor = consumer checkpoint
- Session cursor = replay/resume point

### Event Bus

```typescript
interface IEventBus {
  emit<T>(event: LevEvent<T>): void;
  subscribe<T>(pattern: string, cb: EventCallback<T>): () => void;
  history(filter?: EventFilter): LevEvent[];
}

// Glob pattern matching
bus.subscribe('session.*', handler);      // session.died, session.started
bus.subscribe('idea.**', handler);        // idea.captured, idea.capture.voice
bus.subscribe('*', handler);              // all events
```

---

## Pipeline

```
EventProviders (git, cron, imsg) ──→ LevEvent ──→ events.Bus
Watchers (fs, transcripts)       ──→ LevEvent ──→ events.Bus
                                                      │
                    ┌─────────────────────────────────┼─────────────────────────┐
                    ↓                                 ↓                         ↓
              triggers.Engine                  lifecycle.FSM              events.EventLog
              (match rules)                    (state projection)         (persistence)
                    │
                    ↓
              Action: flowmind.run(path)
                    │
                    ↓
              flowmind.Executor
```

---

## Lifecycle FSMs

### Execution Machine (COMPLETE)
```
pending → running → completed
                  → failed
                  → stalled
```

### Idea Lifecycle (Frontmatter + Events)
```
idle → captured → crystallizing → crystallized → manifesting → completed
```

### Session Lifecycle
```
idle → initializing → active → checkpointing → completing → completed
```

### Synth Lifecycle (Ephemeral → Promoted)
```
ephemeral → promoted → archived
```

**Key Decision**: Simple entities use frontmatter + events + policy (NOT XState).
XState reserved for complex execution tracking only.

---

## Entity States (Ephemeral as STATE)

**All entities transition through states. Ephemeral is a state, not a type.**

```
ephemeral → captured → classified → crystallizing → crystallized → manifesting → completed
    ↓           ↓          ↓            ↓              ↓             ↓            ↓
[pre-commit] [raw]    [typed]     [refining]    [actionable]  [executing]    [done]
                                       ↘ discarded
```

| State | Meaning | Graph Operations |
|-------|---------|------------------|
| ephemeral | Pre-commit, not in graph yet | Proposal pending |
| captured | Raw input, minimal structure | Entity added |
| classified | Entity type determined | Type claim accepted |
| crystallizing | Being refined | Multiple patches |
| crystallized | Actionable, ready for work | Validation gates pass |
| manifesting | Being executed | Links to BD tasks |
| completed | Done | Final patch, archived |
| discarded | Rejected | Entity tombstoned |

---

## Graph Operations

**Core principle: Every change is a graph patch. No file operations — only graph mutations.**

### Operation Types

```typescript
type GraphOperation =
  | { op: 'add_entity'; entity: Entity }
  | { op: 'patch_entity'; entity_id: string; patch: JSONPatch[] }
  | { op: 'remove_entity'; entity_id: string }
  | { op: 'add_claim'; claim: Claim }
  | { op: 'update_truth_state'; claim_id: string; state: TruthState; evidence?: string[] }
  | { op: 'add_link'; from: string; rel: string; to: string }
  | { op: 'remove_link'; from: string; rel: string; to: string };
```

### Link Types

| Link Type | Meaning | Example |
|-----------|---------|---------|
| imports | Direct dependency | module/harness → module/events |
| imported_by | Reverse dependency | module/events ← module/harness |
| side_effects | Non-obvious impact | hook/pre-commit → file/*.ts |
| blocks | Execution dependency | task/T1 → task/T2 |
| blocked_by | Reverse execution dep | task/T2 ← task/T1 |
| produces | Output artifact | task/T1 → artifact/schema.ts |
| consumes | Input dependency | task/T2 → artifact/schema.ts |

---

## LevEvent Schema (Updated)

```typescript
interface LevEvent<T = unknown> {
  id: string;
  type: string;
  source: string;
  time: string;
  data: T;

  // Lev extensions
  level: 'L0' | 'L1' | 'L2' | 'L3';
  aggregateId?: string;
  aggregateType?: 'idea' | 'session' | 'spec' | 'feature' | 'task';
  correlationId?: string;
  causationId?: string;

  // Entity lifecycle state (NEW)
  lifecycleState?: 'ephemeral' | 'captured' | 'classified' | 'crystallizing' |
                   'crystallized' | 'manifesting' | 'completed' | 'discarded';

  // Graph operation (NEW)
  operation?: GraphOperation;
}
```

---

## LOCKED DECISIONS

### Q1: observe/ → events/ ✅
**Decision**: Rename for discoverability
**Package**: `@lev-os/events` (was @lev-os/observe)

### Q2: Single event schema ✅
**Decision**: LevEvent everywhere
**Retire**: LifecycleEvent, Event<T>, CloudEvents (internal)

### Q3: Single event log ✅
**Decision**: `$XDG_DATA_HOME/lev/events.jsonl` with cursors
**Consolidate**: 3 separate JSONL files → 1

### Q4: lifecycle/ scope ✅
**Decision**: FSMs + gates + projections (NO rule engine)
**Rule engine**: Lives in triggers/

### Q5: ValidationGateExecutor location ✅
**Decision**: Lives in `events/` (subscribes to Bus)
**Gap**: ~300 LOC needed

---

## Implementation Status

| Component | Status | Notes |
|-----------|--------|-------|
| EventBus | ✅ | Glob patterns, singleton |
| JsonlPersistence | ✅ | Streaming reads |
| LevEvent schema | ⚠️ | Need to consolidate types |
| ValidationGateExecutor | ❌ | ~300 LOC needed |
| Execution FSM | ✅ | XState v5 |
| Entity FSM | ⏳ | Stub only |
| Frontmatter manager | ✅ | YAML headers |
| Policy engine | ✅ | Promotion rules |

---

## Epics

- lev-cpnv — Lifecycle FSM Consolidation (renamed)
- lev-spj5 — Lifecycle v3 (needs clarification: real or aspirational?)

---

## Migration Path

### Phase 1: Rename
```bash
mv ~/lev/core/observe ~/lev/core/events
# Update all imports: @lev-os/observe → @lev-os/events
```

### Phase 2: Consolidate Schema
- Create LevEvent in events/schema.ts
- Add bridges from old types

### Phase 3: Consolidate Logs
- Single `$XDG_DATA_HOME/lev/events.jsonl`
- Migrate cursors

### Phase 4: ValidationGateExecutor
- ~300 LOC new code
- Subscribe to Bus
- Evaluate gates
- Dispatch actions

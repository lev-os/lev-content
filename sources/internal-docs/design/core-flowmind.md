# 06 - Core: FlowMind, Triggers & Kernel

**Status**: ✅ LOCKED (Session 6)
**Epic**: lev-nw2f (FlowMind compiler), lev-mhzl (FlowMind Kernel Validation)
**Last Updated**: 2026-02-10

---

## Scope

**What this covers:**
- FlowMind compiler (YAML → executable targets)
- Triggers rule engine (event matching + action dispatch)
- Kernel validation layer (ValidationGate execution)

**Modules:**
```
~/lev/core/
├── flowmind/           # Compiler + executor (DECLARATION)
│   ├── src/
│   │   ├── parser.ts       # 845 LOC - YAML → IR
│   │   ├── compiler.ts     # 248 LOC - IR → targets
│   │   ├── executor.ts     # 504 LOC - Runtime execution
│   │   ├── router.ts       # 564 LOC - Complexity routing
│   │   ├── schema.ts       # 701 LOC - Types + ValidationGate
│   │   └── targets/        # 6 compilation targets
│   └── examples/
│       └── session-monitoring.gates.yaml
│
├── triggers/           # Rule engine (EXECUTION)
│   ├── src/
│   │   ├── engine.ts       # Match rules, dispatch actions
│   │   ├── registry.ts     # Fractal discovery
│   │   └── actions/        # flowmind.run, shell.exec, notify
│   └── schema.ts
│
└── events/             # Event bus (SPINE) — see design/core-lifecycle.md
    └── ValidationGateExecutor  # ~300 LOC NEEDED
```

---

## CORE PRINCIPLE (LOCKED)

**FlowMind DECLARES, Triggers EXECUTES, Events is the SPINE.**

| Module | Responsibility | NOT Responsible For |
|--------|---------------|---------------------|
| `flowmind/` | Compiler, executor, IR | Scheduling, event emission |
| `triggers/` | Rule engine, registry, action dispatch | Event storage, state |
| `events/` | Bus, EventLog, ValidationGateExecutor | Rule matching, FSMs |
| `lifecycle/` | FSMs, projections, gates | Rule engine, bus |

---

## FlowMind Pipeline (COMPLETE)

```
INPUT (3 formats)
  │
  ├─→ StepBasedDocument     (*.flow.yaml)
  ├─→ GraphDocument         (nodes + entry)
  └─→ TemplateDocument      (metadata + policies)
                          ↓
PARSER (845 LOC) ✅
  └─→ FlowMindDocument.nodes[]
                          ↓
COMPILER (248 LOC) ✅
  └─→ 6 targets:
      ├─ smartdown.ts   → .smart.md
      ├─ openprose.ts   → .prose.md
      ├─ system-prompt.ts → .system.md
      ├─ prompt-gen.ts  → .prompt.yaml
      ├─ schedule.ts    → .schedule.yaml
      └─ hooks.ts       → .hooks/
                          ↓
ROUTER (564 LOC) ✅
  └─→ 8-factor complexity → claude_code_cli | hybrid | lev_exec
                          ↓
EXECUTOR (504 LOC) ⚠️ PARTIAL
  ├─ executeExec()     ✅
  ├─ executeIntake()   ✅
  ├─ executeParallel() ✅
  ├─ executeNotify()   ❌ TODO
  └─ executeAgent()    ❌ TODO
```

---

## Shift-Left Work Ingress (CANONICAL)

FlowMind stays first in the runtime path. We shift `$work` left by classifying and compiling early:

```
User input
  ↓
Shift-left ingress (prompt guard + deterministic classifier + safety scan)
  ↓
FlowMind IR compile (small executable intent plan)
  ↓
Kernel policy (ValidationGateExecutor at runtime)
  ↓
Route to execution lane:
  - inline action
  - background research worker
  - background task worker
  ↓
events.jsonl + lifecycle projection + artifacts
```

### LLM Call Policy (3 Gates)

| Gate | Purpose | LLM Use |
|------|---------|---------|
| Gate 0 | Fast guard/classifier/safety pass | **No LLM** (deterministic only) |
| Gate 1 | Resolve ambiguous classification/workstream | Cheap model + strict JSON schema |
| Gate 2 | Deep proposal/spec/reasoning | Expensive model only when needed |

Rule: if Gate 0 confidence is high, do not call LLM.

### Large Text Policy

Large payloads do not go directly to reasoning:

```
input > limit
  → chunk (token-bounded windows)
  → map summaries + entity extraction
  → reduce summary + evidence refs
  → compile reduced context to FlowMind IR
```

This keeps runtime deterministic and bounded while preserving traceability.

### Background Process Contract

- `query` / research-like intents default to **background research worker**.
- `task` / execution intents default to **background task worker**.
- `idea` / `memory` / `patch` default to inline capture + lifecycle transition.
- Worker lifecycle is tracked through events (`queued -> running -> completed|failed`) and projected into work states.

---

## ValidationGates (SCHEMA EXISTS, EXECUTOR MISSING)

**Schema defined** at `flowmind/src/schema.ts:640-750`:
```typescript
interface ValidationGate {
  id: string;
  name: string;
  if: GateCondition;      // event, filter, scope, debounce
  then: GateAction[];     // notify, execute, escalate, log, enqueue
  else?: GateAction[];
}
```

**Example** at `flowmind/examples/session-monitoring.gates.yaml`:
```yaml
- id: session-death-notification
  if:
    event: session.died
    scope: project
    filter: "context.project != null"
  then:
    - type: notify
      channels:
        - type: email
          to: "{{context.project.email}}"
```

**GAP**: No `ValidationGateExecutor` exists to evaluate gates at runtime.
- Schema: ✅ 111 LOC defined
- Executor: ❌ ~300 LOC needed
- Location: Should live in `events/` (subscribes to Bus)

---

## Kernel vs Compiler (CLARIFIED)

| Layer | Purpose | When | Status |
|-------|---------|------|--------|
| **Compiler** | YAML → IR → targets | Build time | ✅ COMPLETE |
| **Kernel** | Policy enforcement via ValidationGates | Runtime | ❌ MISSING |

**Architecture**: Compiler = build time, Kernel = run time

**Kernel = ValidationGateExecutor**:
```typescript
class ValidationGateExecutor {
  constructor(eventBus: IEventBus, gates: ValidationGate[])

  // Subscribes to gate.if.event patterns
  // Evaluates gate.if.filter conditions
  // Executes gate.then actions
}
```

### Rust-First Boundary (without breaking FlowMind-first)

- FlowMind IR and schema remain the canonical interface.
- Runtime kernel pieces move first: guard, gate executor, scheduler, worker queue.
- TS execution remains a compatibility adapter until parity gates are green.
- No big-bang rewrite: YAML spec stays stable; implementation backend can be Rust/TS.

---

## Triggers/FlowMind Relationship (LOCKED)

**Current State**: triggers/ and flowmind/ are INDEPENDENT (zero imports between them)

**Locked Decision**: Triggers reads FlowMind YAML directly (no compilation to manifest)

```yaml
# FlowMind declares trigger
# .lev/hooks/session-monitor.flow.yaml
name: session-monitor
trigger:
  on: ["session.died"]
steps:
  - id: notify
    action: tool
    tool: notify
```

```
# Triggers understands trigger: block
triggers.Engine:
  1. Scan .lev/hooks/*.flow.yaml
  2. Extract trigger: declarations
  3. Subscribe to events.Bus
  4. On match → flowmind.run(path)
```

**DELETE**: Legacy daemon scheduler ownership removed — replaced by cron EventProvider + triggers

---

## Implementation Status

| Component | LOC | Status |
|-----------|-----|--------|
| Parser | 845 | ✅ COMPLETE |
| Compiler | 248 | ✅ COMPLETE |
| Router | 564 | ✅ COMPLETE |
| Executor | 504 | ⚠️ PARTIAL (notify/agent TODO) |
| 6 Targets | 1,846 | ✅ COMPLETE |
| ValidationGate Schema | 111 | ✅ COMPLETE |
| ValidationGateExecutor | ~300 | ❌ MISSING |
| triggers/ Integration | ~200 | ❌ MISSING |

**Total Implemented**: ~7,300 LOC
**Total Missing**: ~500 LOC (GateExecutor + triggers integration)

---

## Epics

- lev-nw2f — FlowMind compiler (9 tasks)
- lev-mhzl — FlowMind: Kernel Validation Layer (renamed)
- lev-qo1m.11 — triggers/ decomposition → flowmind/ + plugins/

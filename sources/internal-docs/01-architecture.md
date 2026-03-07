# Lev 1.0 Architecture - CANONICAL REFERENCE

**Created**: 2026-01-26
**Status**: Living Document - Canonical System Design
**Version**: Pre-1.0 (targeting 0.0.1 launch)

---

## Executive Summary: What is Lev?

**Lev is a universal agent runtime.**

Build AI agents once. Deploy to 38+ platforms. Define workflows in YAML. Let Lev handle the execution complexity.

**Not an OS.** Not a framework. Not a chatbot wrapper. It's the execution layer for Software 3.0.

**What makes it sick:**
- Platform adapters (target: 38+ across Claude Code, Clawdbot, Cursor, Windsurf, raw APIs)
- FlowMind compiler (YAML workflows → executable targets)
- Polyglot search (LEANN + ck + mgrep - semantic + fast)
- Lifecycle FSMs (predictable agent behavior via XState)
- Hex architecture (swap LLMs, platforms, storage - core stays unchanged)
- Fractal config (XDG system-wide → project → local overrides)

**Built by**: You + me since last year
**Status**: Code works. Needs polish. Ready for 0.0.1.

---

## LEV + AgentPing Operating Boundary (Current Program Contract)

LEV and AgentPing are intentionally separate layers:

- LEV = runtime/orchestration/policy/worker execution
- AgentPing = human-loop interaction surfaces and action UX
- AgentLease (POC) = protected-action lease + step-up auth gate

North-star execution model:

1. AgentPing captures intent and renders review/approval surfaces.
2. AgentLease grants scoped lease for protected actions.
3. LEV enforces ABAC and runs ephemeral sandbox workers.
4. AgentPing renders progress/results and captures follow-up human decisions.

Companion contract:

- `docs/_archive/04-agentping-human-loop.md` (archived; see `.lev/pm/specs/` for active specs)

---

## Vision & Roadmap

> "The universal agent runtime - build once, deploy to 38+ platforms, define workflows in YAML, let Lev handle execution."

**Target**: Framework builders + power users who need agents everywhere
**Competitors**: LangChain (framework), AutoGen (multi-agent), crewAI (teams), raw SDKs
**Advantage**: Platform-agnostic. Swap LLMs, platforms, storage. Core stays unchanged.

### 3-Year Horizon

**Year 1 (2026): Foundation** — 0.0.1 (Week 1) → 0.1.0 (Month 1) → 1.0.0 (Month 3)
**Year 2 (2027): Ecosystem** — Plugin marketplace, multi-language SDKs (Python, JS, Go, Rust), enterprise features, 1M+ downloads
**Year 3 (2028): Platform** — Hosted runtime, visual workflow builder, agent collaboration protocol, industry standard

### Success Metrics

| Milestone | Users | Platforms | Test Coverage | Other |
|-----------|-------|-----------|---------------|-------|
| 0.0.1 | — | Code works | — | Structures renamed, canonical docs |
| 0.1.0 | 1,000 WAU | 5 live | >70% | 10 example workflows |
| 1.0.0 | 10,000 WAU | 15+ | >90% | Stable API (semver), 3+ enterprise customers |

### Target Core Structure (12 Critical Modules)

```
~/lev/core/
├── agent-adapter/          # Conversation sync runtime (platform adapters moved to plugins/platforms/)
├── harness/                # Execution engine (hex arch)
├── flowmind/               # YAML → executable compiler
├── index/                  # Polyglot search (LEANN+ck+mgrep)
├── lifecycle/              # State machines (XState FSMs)
├── polyglot-runners/       # Cross-language SDK
├── graph/                  # Entity persistence (real orch)
├── triggers/               # Event system
├── cli/                    # lev command router
├── config/                 # XDG fractal config
├── build/                  # lev-dev tooling
└── types/                  # Shared TypeScript interfaces
```

---

## C1: System Context

```
                    ┌─────────────────────────────────────────┐
                    │                                         │
                    │             DEVELOPERS                  │
                    │    (AI Engineers, Solo Devs, Teams)     │
                    │                                         │
                    └──────────────┬──────────────────────────┘
                                   │
                                   │ Use Lev to build agents
                                   │
                    ┌──────────────▼──────────────────────────┐
                    │                                         │
                    │               LEV                       │
                    │      Universal Agent Runtime            │
                    │                                         │
                    │  • FlowMind (YAML workflows)           │
                    │  • Harness (Hex execution)             │
                    │  • Index (Poly search)                 │
                    │  • Lifecycle (FSMs)                    │
                    │  • Config (Fractal)                    │
                    │  • Plugins (First-class)               │
                    │                                         │
                    └──┬──────────┬──────────┬───────────┬───┘
                       │          │          │           │
            ┌──────────▼──┐  ┌───▼────┐  ┌─▼─────┐  ┌─▼──────┐
            │             │  │        │  │       │  │        │
            │  PLATFORMS  │  │  LLMs  │  │ Tools │  │ Memory │
            │             │  │        │  │       │  │        │
            │ Claude Code │  │Claude  │  │Search │  │AutoMem │
            │ Clawdbot    │  │OpenAI  │  │Bash   │  │Graphiti│
            │ Cursor      │  │Gemini  │  │HTTP   │  │AgentFS │
            │ 35 more...  │  │OpenRouter│ │Cron  │  │        │
            │             │  │        │  │       │  │        │
            └─────────────┘  └────────┘  └───────┘  └────────┘
```

**Value Proposition**:
- **For AI Engineers**: Stop rewriting agents for every platform
- **For Solo Devs**: YAML workflows, not Python chains
- **For Teams**: One runtime, 38 deployment targets

---

## C2: Containers (Major Components)

```
┌────────────────────────────────────────────────────────────────┐
│                           LEV CORE                             │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │  FlowMind    │  │   Harness    │  │  Index/Poly      │   │
│  │              │  │              │  │                  │   │
│  │  Compiler    │──▶│  Execution   │  │  LEANN + ck     │   │
│  │  Assembler   │  │  Providers   │  │  + mgrep        │   │
│  │  Decompiler  │  │  Commands    │  │  Semantic       │   │
│  │  YAML IR     │  │  Hex Arch    │  │  Search         │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │  Lifecycle   │  │   Config     │  │  Plugins         │   │
│  │              │  │              │  │                  │   │
│  │  Idea FSM    │  │  Fractal     │  │  Workshop        │   │
│  │  Session FSM │  │  XDG         │  │  Timetravel      │   │
│  │  Synth FSM   │  │  Extends     │  │  AutoMem         │   │
│  │  Events      │  │  Overrides   │  │  Graphiti        │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 1. FlowMind (Declarative Layer)

**Purpose**: YAML workflows → executable targets

**Location**: `~/lev/core/flowmind/`

**Components**:
- **Compiler**: YAML → executable DAG
- **Assembler**: Target-specific output (prompts, hooks, schedules)
- **Decompiler**: Reverse engineer workflows from logs
- **Context Resolver**: PLT/GOT pattern for dynamic loading

**Example**:
```yaml
# workflow.flow.yaml
name: research-agent
complexity: 3
loops:
  - name: search
    agent_prompt: "Find papers on {{topic}}"
    output: papers.json
  - name: analyze
    agent_prompt: "Summarize findings"
    output: summary.md
```

**Status**: Implementation exists, epic lev-nw2f (9 tasks, some open)

---

### 2. Harness (Execution Engine)

**Purpose**: Hex architecture execution with pluggable providers

**Location**: `~/lev/core/harness/`

**Hex Architecture**:
```
         ┌─────────────────────────┐
         │    Domain Core          │
         │  (Business Logic)       │
         └───────────┬─────────────┘
                     │
         ┌───────────▼─────────────┐
         │    Ports (Interfaces)   │
         │  - IHarness             │
         │  - IProvider            │
         │  - IExecutor            │
         └───────────┬─────────────┘
                     │
    ┌────────────────┼────────────────┐
    │                │                │
┌───▼────┐      ┌───▼────┐      ┌───▼────┐
│  CLI   │      │  SDK   │      │  API   │
│Adapter │      │Adapter │      │Adapter │
└────────┘      └────────┘      └────────┘
```

**Providers** (~/lev/core/harness/src/providers/):
- `claude-agent-sdk.ts` - Default (Claude SDK)
- `ai-sdk.ts` - Vercel AI SDK
- `pi.ts` - Raw API calls
- `cli.ts` - Subprocess execution
- `shell.ts` - Direct shell

**Commands** (~/lev/core/harness/src/commands/):
- `exec.ts` - Main entry point
- Registry-based discovery (no magic routing)

**Status**: Production-ready, active development

---

### 3. Index/Poly (Polyglot Search)

**Purpose**: Find anything, anywhere, fast

**Location**: `~/lev/core/index/`, `~/lev/core/polyglot-runners/`

**Three Search Engines**:
1. **LEANN** (embeddings) - Semantic search
2. **ck** (code kernel) - AST-aware code search
3. **mgrep** (ripgrep) - Fast text search

**Integration**:
```typescript
import { search } from '@lev-os/index';

// Finds function across 10 repos in <1s
const results = await search({
  query: 'function login()',
  repos: ['~/lev', '~/clawd', ...],
  engines: ['ck', 'mgrep', 'leann']
});
```

**Status**: 144 tests passing in polyglot-runners

---

### 4. Lifecycle (State Machines)

**Purpose**: Predictable agent behavior via XState FSMs

**Location**: `~/lev/core/lifecycle/`

**Three FSMs**:

**Idea FSM**:
```
ephemeral → captured → classified → crystallizing →
crystallized → manifesting → completed
                       ↘ discarded
```

Post-completion archival is a storage/projection step, not an entity lifecycle state.

**Session FSM**:
```
idle → initialized → running → paused →
resuming → completed → archived
```

**Synth FSM**:
```
idle → analyzing → synthesizing → validating →
complete → archived
```

**EphemeralStore** (Pre-FSM buffer):
```
pending → reviewing → crystallizing → expired
```

**Key Features**:
- XState v5 async actors
- LevEvent format logging
- Pluggable actor implementations (LLM, BD, filesystem)

**Status**: Operational, documented in handoffs

---

### 5. Config (Fractal System)

**Purpose**: XDG-compliant config with extends pattern

**Location**: `~/lev/core/config/`

**Structure**:
```
~/.config/lev/config.yaml           # System-wide
    ↓ extends
~/project/.lev/config.yaml          # Project-specific
    ↓ extends
~/project/.lev/local.config.yaml    # Local overrides (gitignored)
```

**XDG Paths**:
- Config: `~/.config/lev/`
- Data: `~/.local/share/lev/`
- State: `~/.local/state/lev/`
- Cache: `~/.cache/lev/`

**NO ~/.lev pollution** - Legacy paths removed.

**Status**: XDG compliance complete (taxonomy handoff 2026-01-25)

---

### 6. Plugins (First-Class Extensions)

**Purpose**: Extend Lev without forking

**Location**: `~/lev/core/` (integrated), `~/lev/plugins/` (external)

**First-Class Plugins**:
- **workshop** (`~/lev/core/workshop/`) - Agent collaboration patterns
- **timetravel** (research) - Temporal debugging
- **automem** - Memory management (SQLite + embeddings)
- **graphiti** - Neo4j knowledge graphs
- **agentfs** - Virtual filesystem for agents
- **openwork** - Business suite integration
- **skill-seekers** - Dynamic skill discovery

**Plugin Architecture**:
```yaml
# plugin.yaml
name: my-plugin
version: 1.0.0
capabilities:
  - memory
  - search
hooks:
  - pre_execute
  - post_execute
```

**Status**: workshop exists, others in progress

### 7. Coordination (Worker Orchestration)

**Purpose**: FlowMind-driven CLI orchestration for spawning, managing, and observing agent workers

**Location**: FlowMind executor (`~/lev/core/flowmind/src/executor.ts`) + poly registry entries

**Design Doc**: [`core-coordination.md`](file:///Users/jean-patricksmith/digital/leviathan/docs/design/core-coordination.md)

**Pattern**: NTM and OpenCode are CLIs — we orchestrate them with FlowMind YAML, not TypeScript adapter classes. If a tool ships an SDK, it gets a TS adapter in `core/exec/src/adapters/`.

**New FlowMind Step Actions**:
- `spawn` — spawn N workers via CLI backend (ntm/opencode)
- `wait` — block until workers reach a condition (idle/complete/error)

**Existing Steps Used for Coordination**:
- `exec` — run `ntm --robot-send`, `ntm --robot-tail`, etc.
- `parallel` — distribute tasks across workers concurrently
- `agent` — `lev exec --adapter=X` for single-agent dispatch

**Backends** (declared in poly registry, not as TS classes):

| Backend | Binary | Coordinate Via | Dashboard |
|---|---|---|---|
| NTM | `ntm` | `--robot-*` CLI commands | AgentPing (via `on_after` hooks) |
| OpenCode | `opencode` | `serve`/`attach` CLI commands | AgentPing (via `on_after` hooks) |

**Dashboard**: Always AgentPing. Bridge is FlowMind `on_after` lifecycle hooks — no separate bridge class.

**Mothership mapping**: `LEV_MODE=mothership` → FlowMind runs `opencode serve`, `LEV_MODE=satellite` → `opencode attach URL`, standalone → `ntm` locally.

**Status**: Design complete, spec complete, ~75 LOC implementation pending

---

## C3: Component Deep Dive

### Harness - Hexagonal Architecture Detail

**Location**: `~/lev/core/harness/src/`

**Directory Structure**:
```
harness/
├── src/
│   ├── interface.ts          # Port (IHarness, IProvider)
│   ├── providers/            # Adapters (implementations)
│   │   ├── claude-sdk.ts     # Default provider
│   │   ├── ai-sdk.ts
│   │   ├── pi.ts
│   │   ├── cli.ts
│   │   └── shell.ts
│   ├── commands/
│   │   └── exec.ts           # CLI entry point
│   └── agentctl.ts           # Orchestrator class
└── tests/
    └── integration/
```

**Provider Interface**:
```typescript
export interface IProvider {
  readonly name: string;
  init(config: HarnessConfig): Promise<void>;
  run(task: Task): Promise<Result>;
  abort(): void;
  on(event: string, cb: EventCallback): void;
}
```

**Ralph Loop Integration**:
```bash
# External to agent (bash while loop)
while true; do
  lev exec "$PROMPT"  # Fresh context each iteration
  if output contains "<promise>"; then break; fi
  if iteration >= max; then escalate; fi
done
```

**Key Principle**: Ralph loop is EXTERNAL. Agent doesn't know it's looping.

---

### FlowMind - Compiler Detail

**Location**: `~/lev/core/flowmind/src/`

**Directory Structure**:
```
flowmind/
├── src/
│   ├── compiler.ts           # YAML → DAG
│   ├── assembler.ts          # DAG → Targets
│   ├── decompiler.ts         # Logs → YAML
│   ├── context-resolver.ts   # PLT/GOT pattern
│   ├── parser.ts             # YAML parsing
│   └── executor.ts           # Runtime execution
├── schemas/
│   └── entities/             # Entity .flow.yaml files
└── templates/
```

**Compilation Targets**:
1. **Prompts** - System prompt templates
2. **Hooks** - Pre/post execution hooks
3. **Schedules** - Cron-like timing
4. **DAG** - Executable graph
5. **Audit** - LevEvent logging
6. **Config** - Runtime configuration

**Example Compilation**:
```yaml
# Input: workflow.flow.yaml
name: research
loops:
  - name: search
    agent_prompt: "Find {{topic}}"

# Output: Targets
prompts/search.txt          # "You are a research assistant. Find papers on {{topic}}"
hooks/pre_search.ts         # Rate limiting, auth checks
dag/workflow.json           # Topological sort, dependencies
```

---

### Index - Polyglot Search Detail

**Location**: `~/lev/core/index/`, `~/lev/core/polyglot-runners/`

**Three Engines**:

**1. LEANN (Semantic)**:
- Embeddings-based (OpenAI, local models)
- Understands intent: "authentication code" finds login functions
- ~500ms for 10k documents

**2. ck (Code Kernel)**:
- AST-aware parsing
- Finds: function definitions, imports, types
- Language-specific (JS/TS, Python, Go, Rust)
- ~100ms for 1M LOC

**3. mgrep (Ripgrep)**:
- Fast text search (ripgrep wrapper)
- Regex support
- ~50ms for 1M files

**Unified Interface**:
```typescript
search({
  query: 'function login()',
  engines: ['ck', 'mgrep', 'leann'],
  repos: ['~/lev', '~/clawd'],
  filters: { language: 'typescript' }
})
```

**Status**: 144 tests passing, production-ready

---

## C4: Code Patterns

### 1. Hexagonal Architecture (Ports & Adapters)

**Pattern**: Domain logic isolated from infrastructure

**Implementation**:
```
Core Domain (business logic)
    ↓ uses
Ports (interfaces)
    ↓ implemented by
Adapters (concrete implementations)
```

**Example in Harness**:
```typescript
// Port (interface)
export interface IHarness {
  run(task: Task): Promise<Result>;
}

// Adapter (implementation)
export class ClaudeSDKHarness implements IHarness {
  async run(task: Task): Promise<Result> {
    // Claude SDK-specific logic
  }
}
```

**Benefits**:
- Swap adapters without touching core
- Test domain logic in isolation
- Add new platforms without refactoring

---

### 2. XState v5 Async Actors

**Pattern**: State machines with async actors for side effects

**Implementation**:
```typescript
import { createMachine, createActor } from 'xstate';

const ideaMachine = createMachine({
  id: 'idea',
  initial: 'idle',
  states: {
    idle: {
      on: { CAPTURE: 'captured' }
    },
    captured: {
      invoke: {
        src: 'captureIdea',  // Async actor
        onDone: 'crystallizing',
        onError: 'error'
      }
    },
    // ... more states
  }
});

const actor = createActor(ideaMachine);
actor.start();
```

**Benefits**:
- Predictable state transitions
- Explicit error handling
- Testable state logic
- Visual state charts

---

### 3. LevEvent Format

**Pattern**: Standardized event logging for audit trails

**Implementation**:
```typescript
{
  "version": 1,
  "type": "lev.idea.captured",
  "source": "/lev/lifecycle/idea-fsm",
  "ts": "2026-01-26T07:00:00Z",
  "time": "2026-01-26T07:00:00Z",
  "id": "uuid",
  "data": {
    "ideaId": "abc-123",
    "title": "New feature",
    "confidence": 0.85
  }
}
```

**Benefits**:
- Standard format (interop with other tools)
- Replay capability (decompiler uses events)
- Audit trail (compliance-ready)

---

### 4. Fractal Module Resolution

**Pattern**: Config/build resolution with extends chain

**Implementation**:
```typescript
// Resolver walks extends chain
const config = await resolveConfig({
  start: '~/project/.lev/config.yaml',
  searchPaths: [
    '~/project/.lev/',
    '~/.config/lev/',
    '/etc/lev/'
  ]
});

// Result: Merged config from all extends
```

**Benefits**:
- Team + personal config coexist
- Override only what you need
- Scales from toy to enterprise

---

## Plugin Architecture

### First-Class Plugins (Integrated)

**Workshop** (`~/lev/core/workshop/`):
- Agent collaboration patterns
- CDO (Cognitive Diversity Orchestration)
- Team coordination primitives

**AutoMem** (memory system):
- SQLite + embeddings
- Memory search native tool
- Constitutional memory (filtered by ID)

**Graphiti** (knowledge graphs):
- Neo4j integration
- Entity relationships
- Temporal queries

**AgentFS** (virtual filesystem):
- Agents see virtual filesystem
- FUSE integration
- Permission-controlled

**Timetravel** (research):
- Temporal debugging
- Replay agent runs
- Causal analysis

**OpenWork** (business suite):
- CRM/project management
- Cowork-style integrations
- Task/epic management (BD)

**Skill-Seekers** (skill discovery):
- Dynamic skill loading
- Marketplace integration
- Version management

### Plugin Installation

```bash
lev install --plugin=workshop
lev install --plugin=automem
lev install --app=clawdbot  # Platform integration
```

---

---

## Architecture Principles

### 1. Code is Truth

- Beads track intent
- Code is reality
- When they conflict, code wins
- Document what exists, not what's planned

### 2. Explicit > Magic

- No special folder routing
- Commands declare metadata
- No auto-discovery surprises
- Fractal resolution is explicit

### 3. Hex Architecture Everywhere

- Domain isolated from infra
- Swap adapters without core changes
- Test in isolation
- Add platforms without refactoring

### 4. XDG Compliance

- No `~/.lev` pollution
- Standard paths (`~/.config/lev/`, `~/.local/share/lev/`)
- Play nice with system
- Respect user's environment

### 5. Fractal Everything

- Config scales (system → project → local)
- Build scales (global → module → package)
- Modules self-sufficient but composable

### 6. Fail-Fast / No Hidden Functionality

- **No silent fallbacks.** If an adapter, config source, or provider is not found, the system errors immediately with a clear message. It never silently falls through to a built-in or default path that the operator didn't explicitly request.
- **No swallowed exceptions.** Empty `catch {}` blocks are forbidden. Every catch must either re-throw, log and re-throw, or handle the error with an explicit documented reason.
- **No simulated success.** Deploy runtime status must be explicit and truthful: `apply` is `dry_run | executed | failed`, `verify` is `verified | failed`, and `rollback` is `rolled_back | failed`. Never report an executed success for non-executed operations.
- **Explicit over convenient.** If a provider is not configured, fail — don't guess. If a plan shape doesn't match, fail — don't try a second call signature. If credentials are missing, fail — don't skip the check.
- **Operator trust is non-negotiable.** The operator must be able to trust that what the system reports happened actually happened. Hidden fallback paths and silent degradation destroy this trust and are treated as bugs, not features.

---

## Core Modules Map

**Current snapshot (2026-02-13):** `core/` contains 31 top-level directories.

**Execution & Runtime:**
- harness (hex execution engine)
- agent-adapter (platform adapters; migration target is `plugins/platforms/`)
- flowmind (YAML compiler)
- lifecycle (FSMs + gates)
- events (canonical event bus + event log)

**Infrastructure:**
- config (fractal config)
- index (polyglot search)
- polyglot-runners (poly binder + process lifecycle manager)
- daemons (service daemon implementations and migration target for legacy daemon code)
- triggers (event-driven automation)

**Data & Support:**
- memory (hybrid memory backends/orchestrator)
- domain, types, validation
- cli, commands, channels, contexts
- testing, debug, mesh, skills-registry, platform-detection, auth-sniffer

Use `find core -mindepth 1 -maxdepth 1 -type d` for the authoritative current list.

---

## Poly + Daemons Reconciliation (2026-02-13)

This section resolves ownership drift across poly runtime bindings and daemon service ownership.

### Architectural Drivers

- **Modifiability**: daemon lifecycle should be changed once, in one owner.
- **Reliability**: process supervision needs one consistent health/restart path.
- **Operability**: CLI, daemon, and MCP surfaces must be discoverable from one registry.
- **Portability**: typed poly bindings should remain the source for cross-language SDK behavior.

### Current State (Code Reality)

| Path | Current role in code | Drift vs target |
|---|---|---|
| `core/polyglot-runners/` | Registry builder, PMDaemon adapter, daemon lifecycle orchestration, SDK command registry | Correct owner for binding + lifecycle |
| `core/daemon/` | Service daemons (`lev-learner`, `parallel-research`) plus transitional legacy daemon logic to migrate | Decomposition still incomplete |
| `core/index/src/daemon/` | Local HTTP daemon and client fallback logic for search | Service behavior exists outside poly declarations |
| Poly shorthand namespace | Proto-only surface (`trace.proto`) | Not a runtime ownership target |

### Locked Ownership Boundary

- `core/polyglot-runners/` owns platform-level process lifecycle and binding (`poly.sdk`, `poly.daemon`, `poly.cli`, `poly.mcp`).
- `core/daemon/` is the runtime home for daemon applications/services.
- Legacy daemon logic currently outside `core/daemon/` must be migrated there (or into owning plugins) and removed from legacy locations.
- Poly shorthand namespace is reserved for proto/alias usage, not an implementation home for new runtime logic.

### Convergence Rules

1. Any new CLI, daemon, or MCP surface must be declared in `config.yaml` under `poly:` before shipping.
2. Package-level manifests (`package.json` `leviathan`/`mcp`) are descriptive metadata, not the runtime source of truth.
3. New long-running services do not add bespoke lifecycle loops in feature modules; they bind to poly process supervision.
4. If lifecycle code appears outside poly ownership, treat it as reconciliation debt and track migration.

### SDK-First Execution Surface (Transition Lock)

- `@lev-os/exec` is the canonical execution API for module-to-module calls.
- `lev exec`, MCP tools, and daemon-exposed execution entrypoints must behave as thin wrappers over the same SDK contract.
- Wrapper layers are transport-focused only: argument parsing, IO/session wiring, and error presentation. Orchestration/policy logic belongs in SDK core.
- If SDK and CLI behavior diverge for equivalent inputs, treat that as a contract defect.
- Performance expectation for this migration: direct SDK invocation should be faster than CLI invocation for equivalent workloads, because it avoids CLI bootstrap/PTY/process overhead.
- Current state remains transitional in some paths; migration tasks must preserve 1:1 external behavior while moving orchestration into SDK core.

---

## Directory Structure (Current Snapshot)

```
~/lev/
├── apps/                    # Lev client applications (things that ARE lev)
│   ├── desktop/            # macOS/Windows desktop app
│   ├── expo/               # React Native mobile app
│   ├── flutter/            # Flutter mobile app
│   ├── nextjs/             # Next.js web dashboard
│   └── ...
├── community/               # OSS ecosystem packages (things lev PUBLISHES)
│   ├── lev-portable/       # Architectural thinking toolkit (skill)
│   ├── agentguard/         # Git hooks lock/lease pattern (CLI tool)
│   └── agentping/          # Agent-human interaction protocol (tool)
├── core/                    # Framework internals
│   ├── harness/            # Hex execution engine
│   ├── agent-adapter/      # Platform adapter implementation
│   ├── events/             # Canonical event bus + event log
│   ├── flowmind/           # YAML compiler
│   ├── index/              # Polyglot search
│   ├── lifecycle/          # FSMs + gates
│   ├── memory/             # Memory orchestration
│   ├── config/             # Fractal config
│   └── ...
├── plugins/                 # Lev runtime extensions (require leviathan key)
│   ├── timetravel/         # Time-based job scheduling
│   ├── platforms/          # Coding + gateway adapters
│   ├── deploy/             # Opt-in deploy runtime (`levd`) + deploy adapters
│   ├── codex/              # Code analysis
│   └── ...
├── crates/                  # Rust libraries
│   ├── lev-agentfs/        # Agent filesystem
│   └── lev-tui-*/          # Terminal UI components
├── research/                # Pre-production experiments only
├── .lev/                   # Project config + BD
│   ├── config.yaml         # Extends ~/.config/lev/
│   ├── .beads/             # BD task tracking
│   └── pm/                 # Plans, reports, handoffs
└── README.md

~/.config/lev/              # XDG system config
~/.local/share/lev/         # XDG data
~/.local/state/lev/         # XDG state
~/.cache/lev/               # XDG cache
```

**Directory boundaries:**
- `apps/` = things that ARE lev (client applications for the platform)
- `community/` = things lev PUBLISHES (standalone OSS, no lev runtime dep, independent repos as submodules)
- `core/` = framework internals (module exposure contract required)
- `plugins/` = runtime extensions (leviathan manifest in package.json)
- `crates/` = Rust workspace members
- `research/` = experiments only — graduates to `community/` or `plugins/` when production-ready

---

## Taxonomy (LOCKED VERNACULAR)

> Full reference: `docs/design/vernacular.md`

| Term | Meaning |
|------|---------|
| **Harness** | Hex execution engine (domain core) |
| **Provider** | LLM execution backend |
| **Platform** | External system we integrate with (IDE, gateway, agent CLI) |
| **Runner** | Build-time registry unit |

Reactive layer adds 7 more terms: Event, Source, EventProvider, Watcher, Trigger, Action, Bus, EventLog.

---

## Core System Contract & Config

> Full reference: `docs/design/contracts.md`

**7 standards**: SRP, Standalone Value, Zero Sibling Coupling, Config Declaration, XDG Compliance, LevEvent, Vernacular.

**Config resolution** (last wins): System → Project → Module → Environment.

**Refinement passes**: L1 Layout Coherence (IN PROGRESS) → L2 Contract Compliance → L3 Integration Validation.

---

## Status: Where We Are

**Code Reality**:
- Core modules exist (46 directories)
- FlowMind compiler operational
- Polyglot runners: 144 tests passing
- Harness: Hex architecture complete
- Lifecycle: FSMs documented in handoffs
- Config: XDG compliance enforced

**Epic Reality** (beads vs code):
- Some epics OPEN but code exists
- Some tasks DONE but code partial
- **Docs are design truth** - code and events are implementation evidence that must reconcile back to docs

**Launch Readiness**:
- ✅ Core functionality works
- ✅ No catastrophic blockers
- ⚠️ Needs polish (docs, examples, error messages)
- ⚠️ Needs testing (coverage unknown)
- ⚠️ Needs branding (positioning clear, execution needed)

---

## Depth Model & Graph Primitives

> Full reference: `docs/design/graph-primitives.md`

**L0-L3** is a domain-agnostic depth abstraction (Overview → Structure → Details → Runtime). Plugins define domain-specific semantics.

**Graph Primitives (target model):** View, Claim, Evidence, TruthState, Proposal, Gate, Tick. This is ideal-state graph-runtime design; canonical design authority today is `~/lev/docs`.

**Tick Loop:** INGEST → OBSERVE → PROPOSE → GATE → APPLY → UPDATE → EMIT

---

**END OF CANONICAL ARCHITECTURE DOCUMENT**

*This is the canonical system design. Implementation should reconcile to these docs.*

---

## Session Notes & Refinement History

> Moved to `docs/_archive/architecture-session-notes.md`. Q1-Q10 all LOCKED except Q8 (BD epic reconciliation).

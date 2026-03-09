# Core Poly: The Binder Architecture

> **Status:** draft
> **Layer:** L1 Infrastructure
> **Prior Art:** [fractal-polymorph-constitution](_archive/docs/architecture/fractal-polymorph-constitution.md), [core-coordination](core-coordination.md), [memory-poly plan](.lev/pm/plans/memory-poly-architecture-fix-2026-02-01.md)
> **Revision:** 2026-02-13

---

## Purpose

Poly is a **binder** — it takes a component's declarations and wires them into the platform. A component says _what it wants to be_ (daemon, CLI, MCP server, SDK surface), and poly makes it happen.

Poly is NOT:
- ❌ An orchestrator (that's FlowMind / Graph Executor at L2-L3)
- ❌ Business logic (that's the component itself)
- ❌ A framework you inherit from (it's declarative, not OOP)

Poly IS:
- ✅ A binder that reads `poly:` declarations from `config.yaml`
- ✅ A registry builder that merges static + fractal config into a runtime registry
- ✅ A process manager that uses PMDaemon to lifecycle daemons
- ✅ A CLI generator that creates `lev <command>` entry points from declarations

---

## The Binding Model

A component opts in to platform surfaces by declaring them in its `config.yaml`:

```yaml
# core/<component>/config.yaml
package:
  name: my-component
  version: 0.1.0

poly:
  # 1. SDK commands — becomes `lev <command>`
  sdk:
    commands:
      do-thing:
        handler: src/commands/do-thing.ts
        description: "Does the thing"
        args: [...]

  # 2. Daemon — auto-managed by PMDaemon
  daemon:
    entrypoint: src/daemon/server.ts      # file-based routing
    port: 9860
    protocol: http                         # http | grpc
    health: /health
    restart: on-failure

  # 3. MCP server — exposed as MCP tools
  mcp:
    tools:
      my-tool:
        handler: src/mcp/my-tool.ts
        description: "Tool description"
        inputSchema: { ... }

  # 4. CLI — standalone CLI binary
  cli:
    bin: my-cli
    entrypoint: src/cli/index.ts
```

### What Poly Does With Each Declaration

| Declaration | What Poly Does | Implementation |
|---|---|---|
| `poly.sdk` | Registers commands in unified `lev` CLI | `registry-builder.js` → `.build/registry-runtime.yaml` |
| `poly.daemon` | Auto-starts/stops process via PMDaemon | `process-adapter.ts` → `pmdaemon` CLI |
| `poly.mcp` | Registers as MCP tools in adapter | MCP adapter reads from runtime registry |
| `poly.cli` | Generates standalone CLI entry point | Codegen templates in `src/codegen/` |

---

## The Pipeline

```
┌──────────────────────────────────────────────────────────────────┐
│                    BUILD TIME (lev build / lev install)          │
│                                                                  │
│  1. Fractal Scanner                                              │
│     registry-builder.js scans core/*/config.yaml                │
│     Extracts poly: sections from each component                 │
│                                                                  │
│  2. Static Registry                                              │
│     Loads registry.yaml (binaries, daemons, defaults)           │
│                                                                  │
│  3. Merge                                                        │
│     static + discovered → .build/registry-runtime.yaml          │
│                                                                  │
│  4. Project Config (optional)                                    │
│     .lev/config.yaml poly: section for project overrides        │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                    RUNTIME                                       │
│                                                                  │
│  registry-runtime.yaml is the single source of truth for:       │
│  • Which CLI commands exist (lev find, lev exec, lev validate)  │
│  • Which daemons to start (LEANN :50052, ck-lite :9850, etc.)   │
│  • Which MCP tools are available                                 │
│  • Health check endpoints                                        │
│  • Capability mesh for service discovery                         │
└──────────────────────────────────────────────────────────────────┘
```

---

## File-Based Routing

A component can declare a daemon with just a file in the right place:

```
core/<component>/
├── config.yaml                    # poly: declarations
├── src/
│   ├── daemon/                    # Convention: daemon entrypoints here
│   │   └── server.ts              # Auto-discovered if poly.daemon declared
│   ├── commands/                  # Convention: CLI/SDK handlers here
│   │   └── do-thing.ts            # Referenced by poly.sdk.commands
│   └── mcp/                       # Convention: MCP tool handlers here
│       └── my-tool.ts             # Referenced by poly.mcp.tools
```

The `poly.daemon.entrypoint` field can point to:
- A TypeScript file: `src/daemon/server.ts` → built and run via `tsx`/`node`
- A Python module: `python -m my_module.server` → run via `uv run`
- A Go binary: `./bin/my-daemon` → pre-compiled
- An npm script: `npm run daemon` → delegated

PMDaemon manages the process lifecycle (start, stop, restart, health checks).

---

## The Two Daemon Systems — Reconciliation

### Current State (two systems)

| System | Location | Purpose | Process Model |
|---|---|---|---|
| **BD Task Queue** | `core/daemon/` | Polls `.beads/` for tasks, claims/completes them | Single process with timers |
| **Poly Process Manager** | `core/polyglot-runners/src/daemon/` | Worker pool, queue manager, process lifecycle | Multi-process with PMDaemon |

### Target State (one system)

```
core/polyglot-runners/           ← THE binder (poly)
├── src/daemon/                  ← Process manager (core.ts, queue-manager.ts, worker-pool.ts)
├── src/adapters/                ← PMDaemon adapter, process adapter
├── src/registry-builder.js      ← Fractal config scanner
├── registry.yaml                ← Static daemon/binary definitions
└── .build/registry-runtime.yaml ← Merged output

core/daemon/task-queue/         ← Daemon app binds to poly
├── config.yaml                  ← NEW: declares poly.daemon (BD queue as a service)
├── src/daemon-core.ts           ← BD queue logic (business logic stays here)
├── src/queue.ts                 ← Task fetching/claiming
└── src/events.ts                ← Event emission
```

**The reconciliation:**
1. `core/daemon/task-queue/` keeps its BD task queue business logic
2. `core/daemon/task-queue/` adds a `poly.daemon` declaration in its `config.yaml`
3. Poly's process manager (PMDaemon) handles start/stop/restart
4. daemon apps no longer manage their own PID file, heartbeat, or signal handlers — poly does that
5. The daemon worker pool in `core/polyglot-runners/src/daemon/` is the platform-level process manager
6. `core/daemon/task-queue/` is the BD-specific application that runs INSIDE a poly-managed process

```yaml
# core/daemon/task-queue/config.yaml (NEW)
package:
  name: task-queue
  version: 0.1.0
  description: BD-based task queue and FlowMind executor

poly:
  daemon:
    entrypoint: src/daemon-core.ts
    port: 9880
    protocol: http
    health: /health
    restart: on-failure
    autostart: false              # Only starts when BD integration is active
  sdk:
    commands:
      daemon:
        handler: src/cli.ts
        description: "Manage BD task queue daemon"
        args:
          - name: action
            type: string
            enum: [start, stop, status, queue, sync]
```

---

## Existing Poly Declarations (Current State)

These components already declare `poly:` sections:

| Component | `poly.sdk` | `poly.daemon` | Notes |
|---|---|---|---|
| **harness** | `exec` | — | Full command spec with adapter matrix |
| **index** | `find`, `refresh`, `embed`, `batch_embed` | — | gRPC handlers for embed commands |
| **validation** | `validate`, `validate-*` (4 commands) | — | Multi-framework validation |
| **memory** | — | `attach_to: python-daemon` | Attaches to shared Python daemon :9851 |
| **config** | — | — | Declares poly but no commands yet |
| **triggers** | — | — | Declares poly but no commands yet |
| **agent-adapter** | — | — | Consumer of `poly:binaries` fractal |
| **polyglot-runners** | (self) | (self) | Meta: manages the registry itself |

---

## `poly.daemon` Modes

Components can declare daemons in three ways:

### Mode 1: Own Process
Spins up its own daemon process:
```yaml
poly:
  daemon:
    entrypoint: src/daemon/server.ts
    port: 9860
    protocol: http
```

### Mode 2: Attach to Shared Daemon
Registers as a provider inside an existing language daemon:
```yaml
poly:
  daemon:
    attach_to: python-daemon    # Runs inside :9851
    port: 9851
    provider: my-provider
```

### Mode 3: No Daemon
Pure CLI/SDK — no long-running process:
```yaml
poly:
  sdk:
    commands:
      my-command:
        handler: src/commands/my-command.ts
```

---

## SDK Binding

When a component declares `poly.sdk.commands`, the registry-builder adds the command to the unified `lev` CLI. At runtime:

```
User runs: lev validate --target "claim" --frameworks mathematical

1. lev CLI looks up "validate" in registry-runtime.yaml
2. Finds: handler = core/validation/src/commands/validate.js
3. Resolves handler path relative to repo root
4. Executes: node core/validation/src/commands/validate.js --target "claim" --frameworks mathematical
```

For gRPC-backed commands (like `lev embed`):
```
User runs: lev embed --text "hello world"

1. lev CLI looks up "embed" in registry-runtime.yaml
2. Finds: handler = grpc://localhost:50052, proto = VectorIndexService/Embed
3. Ensures LEANN daemon is running (starts via PMDaemon if needed)
4. Makes gRPC call to :50052
```

---

## Cross-References

| Doc | Relationship |
|---|---|
| [01-architecture.md §3](file:///Users/jean-patricksmith/digital/leviathan/docs/01-architecture.md) | Parent — poly is subsystem 3 (Index/Poly) |
| [core-coordination.md](file:///Users/jean-patricksmith/digital/leviathan/docs/design/core-coordination.md) | Sibling — coordination uses poly registry for backend discovery |
| [fractal-polymorph-constitution](_archive/docs/architecture/fractal-polymorph-constitution.md) | Ancestor — poly is Layer 1 in the 4-layer model |
| [registry.yaml](file:///Users/jean-patricksmith/digital/leviathan/core/polyglot-runners/registry.yaml) | Static registry — merged with fractal discoveries |
| [registry-builder.js](file:///Users/jean-patricksmith/digital/leviathan/core/polyglot-runners/src/registry-builder.js) | Implementation — scans config.yamls, outputs runtime registry |

---

## Deferred Chores (Separate Work Items)

These issues were found during the daemon audit but are separate from the poly design:

| Chore | Severity | Description |
|---|---|---|
| `worktree-*.js` missing | 🔴 Crash | `core/commands/src/index.js` exports modules that don't exist |
| `KINGLY_*` env vars | 🟡 Legacy | Should be `LEV_*` throughout `core/debug/`, `core/commands/` |
| `@lev/debug` incomplete | 🟡 Partial | `tracer.js`/`monitor.js` missing — stub or remove |
| Port conflicts in registry | 🟡 Config | 9850 and 9852 claimed by overlapping entries |
| `core/lib/watch/` missing | ℹ️ Note | `collection-watcher.js` depends on missing watcher module |

---

*This document defines how poly binds components to the Leviathan platform. When a component wants to be a daemon, CLI, MCP server, or SDK surface — it declares it in `config.yaml` and poly makes it happen.*

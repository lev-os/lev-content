# Core Coordination Design

> **Status:** draft
> **Layer:** Core Services
> **Prior Art:** [distributed-inference-network.md](file:///Users/jean-patricksmith/digital/leviathan/_archive/docs/architecture/distributed-inference-network.md), [spec-ntm-agentping-remote-workers.md](file:///Users/jean-patricksmith/digital/leviathan/docs/specs/spec-ntm-agentping-remote-workers.md)
> **Revision:** 2026-02-12 — Changed from TypeScript adapter classes to FlowMind-driven CLI orchestration

---

## Purpose

Define Lev's coordination layer for spawning, managing, and observing agent workers. **Coordination is FlowMind YAML, not TypeScript adapter code.** NTM and OpenCode are CLIs — we orchestrate CLIs with FlowMind, we don't wrap them.

### Decision Rule

| Signal | Approach | Example |
|---|---|---|
| Ships SDK/library we'd `import` | TS adapter in `core/exec/src/adapters/` | claude-agent-sdk, ai-sdk |
| Is a CLI binary | FlowMind YAML orchestration | ntm, opencode, codex |
| Needs tight type integration | TS adapter | future `LevCoordination` native gRPC |
| Just needs spawn/wait/tail | FlowMind steps | ntm robot commands |

---

## Architecture

```
                    ┌───────────────────────────────┐
                    │       FlowMind Executor        │
                    │   (core/flowmind/src/executor) │
                    └──────────────┬────────────────┘
                                   │
              Step actions: exec, agent, parallel,
                           spawn (new), wait (new)
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                     │
      ┌───────▼──────┐    ┌───────▼──────┐    ┌────────▼───────┐
      │  ntm (CLI)   │    │ opencode     │    │  lev exec      │
      │              │    │  (CLI)       │    │  (TS adapters)  │
      │ --robot-*    │    │ serve/attach │    │ claude-sdk,     │
      │              │    │              │    │ ai-sdk, shell   │
      └──────────────┘    └──────────────┘    └────────────────┘
              │                    │                     │
              └────────────────────┼────────────────────┘
                                   │
                    ┌──────────────▼────────────────┐
                    │    AgentPing Bridge             │
                    │  (FlowMind on_after hooks)     │
                    └───────────────────────────────┘
```

---

## FlowMind Step Actions

### Existing (working today)

| Action | What it does | Implementation |
|---|---|---|
| `exec` | Runs shell commands | `spawnSync('/bin/bash', ['-c', command])` |
| `agent` | Calls `lev exec --adapter=X` | Resolves `lev` binary, spawns with CLI flags |
| `parallel` | Runs nested steps concurrently | `Promise.all(steps.map(executeStep))` |
| `intake` | Queues data to intake JSONL | XDG-compliant file append |
| `notify` | Sends notification | Queue + EventEmitter |

### New (coordination extensions)

| Action | What it does | Resolves to |
|---|---|---|
| `spawn` | Spawn N workers in a session | `ntm --robot-spawn=SESSION --spawn-cc=N --json` or `opencode serve` |
| `wait` | Block until condition met | `ntm --robot-wait=SESSION --wait-until=CONDITION --json` or poll |

Both are thin wrappers — they construct CLI commands from step config and delegate to `executeExec()`.

---

## Poly Registry Entries

NTM and OpenCode are declared as services in [registry.yaml](file:///Users/jean-patricksmith/digital/leviathan/core/polyglot-runners/registry.yaml), not as TypeScript classes:

```yaml
# registry.yaml additions
ntm:
  binary: ntm
  description: "Named Tmux Manager — headless agent sessions"
  mode: 1                     # simple service exposure
  commands:
    spawn: "--robot-spawn={{session}} --spawn-cc={{count}} --json"
    send:  "--robot-send={{session}} --panes={{pane}} --msg='{{prompt}}' --json"
    wait:  "--robot-wait={{session}} --wait-until={{condition}} --json"
    tail:  "--robot-tail={{session}} --lines={{lines}} --json"
    kill:  "kill {{session}}"

opencode:
  binary: opencode
  description: "OpenCode agent runtime — serve/attach/execute"
  mode: 1
  commands:
    serve:   "serve --port={{port}} --hostname={{hostname}}"
    attach:  "attach {{url}}"
    execute: "{{prompt}}"     # stdin pipe
```

---

## FlowMind Coordination Schema

The `CoordinationAdapter` TypeScript interface becomes a **FlowMind schema extension**:

```yaml
# Coordination step schema (extends flowmind/src/schema.ts)
step_actions:
  spawn:
    required: [backend, count]
    optional: [workdir, env, depth, session_id]
    backends: [ntm, opencode]   # resolved via poly registry

  wait:
    required: [session, condition]
    optional: [timeout_ms, poll_interval_ms]
    conditions: [idle, complete, error]
```

---

## Example: Full Coordination Workflow

```yaml
# .lev/hooks/coordinate-workers.flow.yaml
name: coordinate-epic
description: Distribute epic tasks across NTM workers
trigger:
  on: [exec.epic]
policy:
  determinism: true
  timeout_ms: 600000

steps:
  - id: spawn-workers
    action: spawn
    backend: ntm
    count: 3
    workdir: "{{rootPath}}"
    output: session

  - id: distribute-tasks
    action: parallel
    steps:
      - id: worker-0
        action: exec
        command: "ntm --robot-send={{session}} --panes=0 --msg='{{task_0}}' --json"
      - id: worker-1
        action: exec
        command: "ntm --robot-send={{session}} --panes=1 --msg='{{task_1}}' --json"
      - id: worker-2
        action: exec
        command: "ntm --robot-send={{session}} --panes=2 --msg='{{task_2}}' --json"

  - id: wait-all-idle
    action: wait
    session: "{{session}}"
    condition: idle
    timeout_ms: 600000

  - id: collect-output
    action: exec
    command: "ntm --robot-tail={{session}} --lines=500 --json"
    output: results
    on_after:
      - action: exec
        value: "curl -s http://localhost:7890/api/v1/pings -d '{\"type\":\"notification\",\"level\":\"success\",\"title\":\"Epic complete\"}'"

  - id: cleanup
    action: exec
    command: "ntm kill {{session}}"
```

---

## Dashboard Strategy

**AgentPing is always the dashboard.** Coordination events flow through FlowMind `on_after` lifecycle hooks:

```yaml
on_after:
  - action: exec
    value: "curl -s http://localhost:7890/api/v1/pings -d '...'"
```

The `spec-ntm-agentping-remote-workers` features (F1-F6) remain valid — they're just implemented as FlowMind hooks, not a TypeScript bridge class.

---

## Mothership / Satellite Mapping

| Mode | Config | FlowMind Behavior |
|---|---|---|
| `mothership` | `LEV_MODE=mothership` | Workflow runs `opencode serve`. Accepts satellite connections. |
| `satellite` | `LEV_MOTHERSHIP_URL=http://...` | Workflow runs `opencode attach URL`. Receives work. |
| `standalone` | (default) | Workflow runs `ntm` locally. No serve/attach. |

---

## Fail-Fast Rules (per Principle 6)

1. FlowMind executor already throws on non-zero exit — no silent fallbacks
2. `spawn` step fails if binary not found → step status = `failed`, workflow stops
3. `wait` step returns `timedOut: true` if timeout exceeded — never silently resolves
4. No auto-selection between backends — workflow declares which one explicitly

---

## Cross-References

| Doc | Relationship |
|---|---|
| [01-architecture.md §7](file:///Users/jean-patricksmith/digital/leviathan/docs/01-architecture.md) | Parent. Coordination section. |
| [contracts.md](file:///Users/jean-patricksmith/digital/leviathan/docs/design/contracts.md) | Contract table. `coordination` entry. |
| [core-flowmind.md](file:///Users/jean-patricksmith/digital/leviathan/docs/design/core-flowmind.md) | FlowMind executor — where spawn/wait actions are implemented. |
| [spec-ntm-agentping-remote-workers.md](file:///Users/jean-patricksmith/digital/leviathan/docs/specs/spec-ntm-agentping-remote-workers.md) | AgentPing bridge — implemented as FlowMind on_after hooks. |
| [spec-coordination-adapters.md](file:///Users/jean-patricksmith/digital/leviathan/docs/specs/spec-coordination-adapters.md) | Implementation spec for spawn/wait steps + registry entries. |
| [registry.yaml](file:///Users/jean-patricksmith/digital/leviathan/core/polyglot-runners/registry.yaml) | NTM/OpenCode declared here. |

---
id: spec-coordination-adapters
title: FlowMind Coordination Steps
status: draft
created: 2026-02-12
updated: 2026-02-12
related: [spec-ntm-agentping-remote-workers.md]
revision: "2026-02-12 — Changed from TS adapter classes to FlowMind step actions"
---

# Spec: FlowMind Coordination Steps

**Status:** draft
**Layer:** Core Services
**Priority:** P1
**Design Doc:** [core-coordination.md](file:///Users/jean-patricksmith/digital/leviathan/docs/design/core-coordination.md)

---

## Executive Summary

Add `spawn` and `wait` as first-class FlowMind step actions. Declare NTM and OpenCode as poly registry entries. No new TypeScript classes — coordination is YAML orchestration of CLIs.

---

## Scope

### In Scope

- FlowMind `spawn` step action — spawn N workers via CLI backend
- FlowMind `wait` step action — block until workers reach a condition
- Poly registry entries for `ntm` and `opencode` with command templates
- AgentPing bridge via FlowMind `on_after` lifecycle hooks

### Out of Scope

- TypeScript adapter classes (not needed — these are CLIs)
- Native `LevCoordination` gRPC (future — would be a TS adapter when it ships an SDK)
- P2P mesh / libp2p

---

## Features

### F1: `spawn` Step Action

Spawns N workers in a session. The backend determines which CLI to call.

```yaml
- id: start-session
  action: spawn
  backend: ntm        # resolved via poly registry
  count: 3
  workdir: "{{rootPath}}"
  output: session
```

**Implementation in `executor.ts`:**

```typescript
private async executeSpawn(step: FlowmindStep): Promise<string> {
  const backend = step.data?.backend ?? 'ntm';
  const count = step.data?.count ?? 1;
  const workdir = step.cwd ?? this.context.cwd;
  const sessionId = `lev-${backend}-${Date.now()}`;

  // Resolve command from poly registry (or hardcoded for now)
  const commands: Record<string, string> = {
    ntm: `ntm --robot-spawn=${sessionId} --spawn-cc=${count} --spawn-no-user --spawn-dir=${workdir} --json`,
    opencode: `opencode serve --port=0`,  // auto-port
  };

  const cmd = commands[backend];
  if (!cmd) throw new Error(`Unknown coordination backend: ${backend}`);

  const output = await this.executeExec({ ...step, command: cmd });
  return sessionId;  // stored via step.output variable
}
```

**~30 lines of code.** Delegates to existing `executeExec()`.

### F2: `wait` Step Action

Blocks until workers reach a condition. Polls at configurable interval.

```yaml
- id: wait-idle
  action: wait
  session: "{{session}}"
  condition: idle
  timeout_ms: 600000
```

**Implementation in `executor.ts`:**

```typescript
private async executeWait(step: FlowmindStep): Promise<string> {
  const session = this.substituteVariables(step.data?.session ?? '');
  const condition = step.data?.condition ?? 'idle';
  const timeoutMs = step.data?.timeout_ms ?? 300_000;
  const pollMs = step.data?.poll_interval_ms ?? 2000;

  const cmd = `ntm --robot-wait=${session} --wait-until=${condition} --json`;

  const deadline = Date.now() + timeoutMs;
  while (Date.now() < deadline) {
    try {
      const output = await this.executeExec({ ...step, command: cmd });
      const result = JSON.parse(output);
      if (result.status === condition) return output;
    } catch { /* poll again */ }
    await new Promise(r => setTimeout(r, pollMs));
  }

  throw new Error(`Wait timed out after ${timeoutMs}ms for condition: ${condition}`);
}
```

**~25 lines of code.** Polls existing CLI, throws on timeout (fail-fast).

### F3: Poly Registry Entries

**File:** [registry.yaml](file:///Users/jean-patricksmith/digital/leviathan/core/polyglot-runners/registry.yaml)

```yaml
ntm:
  binary: ntm
  description: "Named Tmux Manager — headless agent sessions"
  mode: 1
  commands:
    spawn: "--robot-spawn={{session}} --spawn-cc={{count}} --json"
    send:  "--robot-send={{session}} --panes={{pane}} --msg='{{prompt}}' --json"
    wait:  "--robot-wait={{session}} --wait-until={{condition}} --json"
    tail:  "--robot-tail={{session}} --lines={{lines}} --json"
    kill:  "kill {{session}}"

opencode:
  binary: opencode
  description: "OpenCode agent runtime"
  mode: 1
  commands:
    serve:   "serve --port={{port}} --hostname={{hostname}}"
    attach:  "attach {{url}}"
    execute: "{{prompt}}"
```

### F4: AgentPing Bridge via Lifecycle Hooks

No bridge class needed. FlowMind `on_after` hooks emit pings:

```yaml
- id: send-task
  action: exec
  command: "ntm --robot-send={{session}} --panes=0 --msg='{{prompt}}' --json"
  on_after:
    - action: exec
      value: >
        curl -s http://localhost:7890/api/v1/pings
        -H 'Content-Type: application/json'
        -d '{"type":"notification","level":"info","title":"Task sent to worker 0"}'
```

---

## File Layout

```
core/flowmind/src/
├── executor.ts              # +spawn, +wait step actions (~55 LOC)
└── schema.ts                # +spawn, +wait type declarations (~20 LOC)

core/polyglot-runners/
└── registry.yaml            # +ntm, +opencode entries
```

**Total new code: ~75 lines.** Compare to original spec's ~500+ lines of TypeScript adapter classes.

---

## Breaking Changes

None. New step actions are additive. Existing `exec`, `agent`, `parallel` steps unchanged.

---

## Acceptance Criteria

1. `spawn` step creates NTM session, returns session ID in output variable
2. `wait` step blocks until condition met or throws on timeout
3. NTM and OpenCode in poly registry with command templates
4. Full coordination workflow (spawn → distribute → wait → collect → cleanup) executes E2E
5. AgentPing pings visible on dashboard via `on_after` hooks
6. No new TypeScript classes — coordination is YAML + ~75 LOC in executor

---

## Test Plan

### Unit Tests (in `core/flowmind/__tests__/`)

| Test | Assertion |
|---|---|
| spawn creates session | Returns session ID, `ntm --robot-spawn` called |
| spawn unknown backend throws | `Error: Unknown coordination backend` |
| wait returns on condition | Resolves when poll returns matching status |
| wait timeout throws | `Error: Wait timed out` after deadline |
| spawn + wait + exec E2E | Full workflow executes all steps in order |

### Integration Test

```bash
# Requires ntm installed
bun -e "
  import { FlowmindExecutor } from './core/flowmind/src/executor.ts';
  const executor = new FlowmindExecutor({ cwd: process.cwd() });
  const result = await executor.execute({
    name: 'test-coordination',
    steps: [
      { id: 'spawn', action: 'spawn', data: { backend: 'ntm', count: 1 } },
      { id: 'wait', action: 'wait', data: { session: '{{spawn}}', condition: 'idle', timeout_ms: 10000 } },
    ]
  });
  console.log(result);
"
```

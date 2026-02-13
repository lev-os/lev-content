---
id: spec-ntm-agentping-remote-workers
title: NTM x AgentPing Remote Worker Integration
status: crystallized
group: agentping
related: [spec-unify-agentping-surfaces.md, spec-coordination-adapters.md]
---

# Spec: NTM x AgentPing Remote Worker Integration

**Status:** crystallized
**Layer:** Services (Space Plan)
**Priority:** P2
**Epic:** ntm-ap (to be created in BD)

> [!IMPORTANT]
> **Rebase Note (2026-02-12):** This spec's bridge hooks (F1-F6) should be implemented at the `CoordinationAdapter` interface level, not NTM-specific. See [`core-coordination.md`](file:///Users/jean-patricksmith/digital/leviathan/docs/design/core-coordination.md) for the interface contract and [`spec-coordination-adapters.md`](file:///Users/jean-patricksmith/digital/leviathan/docs/specs/spec-coordination-adapters.md) for the adapter spec. The `NTMAgentPingBridge` becomes `CoordinationBridge` — all features below remain valid but operate at the interface level so they work for any adapter (NTM, OpenCode, future Lev native).

---

## Executive Summary

Wire NTM headless worker sessions into the AgentPing dashboard for real-time visibility, human-in-the-loop gates, and crash recovery. Six features transform NTM from a blind subprocess executor into an observable, controllable distributed worker system.

## Context

### Existing State

- **NTM Adapter** (`core/harness/src/providers/ntm.ts`): Single-task execution via ntm robot CLI. Spawns session, sends prompt, waits idle, tails output, kills session.
- **NTM Pool** (`core/harness/src/providers/ntm-pool.ts`): Wave-based parallel execution. One session, N panes, round-robin assignment.
- **AgentPing** (`apps/agentping/`): HTTP API on `:7890`, Web UI on `:7891`, WebSocket at `/api/v1/ws`. 10 ping types, enrichment directives, long-poll wait.
- **No bridge exists.** NTM workers execute in silence. No visibility into progress, no human intervention points, no crash recovery UX.
- **CoordinationAdapter** ([design doc](file:///Users/jean-patricksmith/digital/leviathan/docs/design/core-coordination.md)): Interface-level coordination contract. Bridge hooks should emit from here, not from NTM directly.

### Target State

NTM workers emit structured pings to AgentPing at every lifecycle transition. Humans see real-time worker status on the dashboard, can approve/reject execution plans, and intervene on crashes. The dashboard becomes the control plane for remote worker orchestration.

---

## Features

### F1: Worker Lifecycle Pings

**Given** an NTM session spawns via `execute()` or `pool.initialize()`
**When** the session transitions through lifecycle stages
**Then** a notification ping is emitted to AgentPing for each transition

| Event             | Ping Type      | Level     | Payload                                            |
| ----------------- | -------------- | --------- | -------------------------------------------------- |
| Session spawn     | `notification` | `info`    | sessionId, workerCount, workspace, depth           |
| Task sent to pane | `notification` | `info`    | taskId, paneIndex, promptPreview (first 200 chars) |
| Task complete     | `notification` | `success` | taskId, paneIndex, durationMs, promiseDetected     |
| Task error        | `notification` | `error`   | taskId, paneIndex, error message                   |
| Session shutdown  | `notification` | `info`    | sessionId, totalTasks, totalDuration, successRate  |

### F2: Wave Progress Dashboard

**Given** an epic DAG executes via `executeEpicWithPool()`
**When** waves execute sequentially
**Then** the dashboard shows wave-by-wave progress

| Event         | Ping Type                | Payload                                         |
| ------------- | ------------------------ | ----------------------------------------------- |
| Epic start    | `notification` (info)    | epicId, totalWaves, totalTasks, workerCount     |
| Wave start    | `notification` (info)    | waveIndex, waveSize, taskIds                    |
| Wave complete | `notification` (success) | waveIndex, results summary, failedTasks         |
| Epic complete | `notification` (success) | epicId, totalDuration, successRate, failedTasks |
| Epic failed   | `notification` (error)   | epicId, failedWave, failedTasks, error          |

### F3: Human-in-the-Loop (HITL) Gates

**Given** `hitl: true` option is passed to execution config
**When** execution reaches a gate point
**Then** a blocking ping is created and execution pauses until human responds

| Gate              | Ping Type       | Trigger                           | Response Action                           |
| ----------------- | --------------- | --------------------------------- | ----------------------------------------- |
| Pre-epic approval | `step_approval` | Before first wave                 | approve all/partial waves, add directives |
| Depth escalation  | `approval`      | When `LEV_EXEC_DEPTH >= 2`        | approve/deny inception                    |
| Crash recovery    | `selection`     | Worker error in wave              | retry / skip / abort options              |
| Cost threshold    | `approval`      | When estimated tokens > threshold | approve/deny continuation                 |

**Long-poll pattern:** Worker calls `POST /api/v1/pings` then `GET /api/v1/pings/{id}/wait?timeout=300` (5 min max). If timeout → abort with "human unresponsive" error.

### F4: Worker Status Events (Structured)

**Given** the bridge is initialized
**When** any worker state changes
**Then** a structured status object is attached to pings as metadata

```typescript
interface WorkerStatusMeta {
  sessionId: string
  epicId?: string
  workerCount: number
  workers: Array<{
    paneIndex: number
    state: 'idle' | 'busy' | 'error' | 'shutdown'
    currentTaskId?: string
    taskStartedAt?: number
  }>
  depth: number
  wave?: { current: number; total: number }
}
```

### F5: Inception Depth Visualization

**Given** a worker at depth N spawns a sub-worker at depth N+1
**When** the sub-worker registers with AgentPing
**Then** the agentId encodes the depth chain

Format: `lev-ntm-L{depth}-{sessionId}`

Example chain visible in dashboard:

```
lev-ntm-L0-main-session     (human-initiated)
  lev-ntm-L1-sub-abc123     (worker-spawned)
    lev-ntm-L2-sub-def456   (sub-worker-spawned)
```

### F6: Error Recovery Flow

**Given** a worker pane crashes or times out during wave execution
**When** the pool detects the failure
**Then** a selection ping offers recovery options

```json
{
  "type": "selection",
  "title": "Worker 3 failed: task 'build-frontend'",
  "context": "Wave 2/4, 2 of 3 tasks succeeded. Error: idle timeout after 300 polls",
  "options": [
    { "id": "retry", "label": "Retry task", "description": "Re-send to an idle worker" },
    { "id": "skip", "label": "Skip task", "description": "Mark failed, continue to wave 3" },
    { "id": "abort", "label": "Abort epic", "description": "Stop execution, clean up all workers" },
    { "id": "reassign", "label": "Reassign to new pane", "description": "Spawn fresh worker, retry" }
  ],
  "allowMultiple": false
}
```

---

## Contract

### New File: `core/harness/src/providers/ntm-agentping.ts`

```typescript
export interface NTMAgentPingConfig {
  agentpingUrl?: string // default: http://localhost:7890
  agentId?: string // default: lev-ntm-orchestrator
  sessionId?: string // auto-generated from ntm session
  hitl?: boolean // default: false
  costThreshold?: number // tokens before requiring approval
  enabled?: boolean // default: true (degrades gracefully if daemon offline)
}

export class NTMAgentPingBridge {
  constructor(config?: NTMAgentPingConfig)
  async isAvailable(): Promise<boolean>

  // F1: Lifecycle pings
  async onSessionSpawn(meta: WorkerStatusMeta): Promise<void>
  async onTaskSent(taskId: string, paneIndex: number, promptPreview: string): Promise<void>
  async onTaskComplete(taskId: string, paneIndex: number, result: WaveTaskResult): Promise<void>
  async onTaskError(taskId: string, paneIndex: number, error: string): Promise<void>
  async onSessionShutdown(summary: ExecutionSummary): Promise<void>

  // F2: Wave progress
  async onEpicStart(epicId: string, waves: number, tasks: number, workers: number): Promise<void>
  async onWaveStart(waveIndex: number, taskIds: string[]): Promise<void>
  async onWaveComplete(waveIndex: number, results: WaveTaskResult[]): Promise<void>
  async onEpicComplete(epicId: string, summary: ExecutionSummary): Promise<void>

  // F3: HITL gates (blocking — returns human response)
  async requestEpicApproval(epicId: string, waves: WavePlan[]): Promise<ApprovalResult>
  async requestDepthApproval(currentDepth: number, taskDescription: string): Promise<boolean>
  async requestCrashRecovery(taskId: string, error: string, context: string): Promise<RecoveryAction>

  // Internal
  private async sendPing(payload: AgentPingPayload): Promise<string>
  private async waitForResponse(pingId: string, timeoutMs?: number): Promise<AgentPingResponse>
}

export type RecoveryAction = 'retry' | 'skip' | 'abort' | 'reassign'
```

### Modified Files

| File               | Changes                                                                                                                                                                          |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ntm.ts`           | Add optional `bridge?: NTMAgentPingBridge` to config. Call `bridge.onSessionSpawn()`, `bridge.onTaskComplete()`, etc. in `execute()` lifecycle.                                  |
| `ntm-pool.ts`      | Add bridge to pool config. Call `bridge.onEpicStart()`, `bridge.onWaveStart()`, etc. in `executeWave()`. Call `bridge.requestCrashRecovery()` on pane failure when `hitl: true`. |
| `orchestration.ts` | Pass bridge config through to `executeEpicWithPool()`.                                                                                                                           |
| `exec.ts`          | Add `--agentping` and `--hitl` CLI flags. Instantiate bridge when flags present.                                                                                                 |

### Dependencies

- `node-fetch` or native `fetch` (Node 18+ built-in) for HTTP calls to AgentPing
- No new npm packages required (uses fetch API)

### Breaking Changes

None. Bridge is opt-in via config. All existing behavior unchanged when bridge is not provided.

---

## Implementation Guidance

### Recommended Skills

- `arch` — Systems architecture for bridge design
- `agentping` — AgentPing API integration patterns

### Team Structure (kingly-clawd)

| Workstream                | Agent             | Files                                            | Depends On |
| ------------------------- | ----------------- | ------------------------------------------------ | ---------- |
| WS-1: Bridge module       | `general-purpose` | `ntm-agentping.ts`                               | —          |
| WS-2: Adapter hooks       | `general-purpose` | `ntm.ts`, `ntm-pool.ts`                          | WS-1       |
| WS-3: HITL gates          | `general-purpose` | `ntm-agentping.ts` (HITL methods), `ntm-pool.ts` | WS-1       |
| WS-4: CLI + orchestration | `general-purpose` | `exec.ts`, `orchestration.ts`                    | WS-2, WS-3 |
| WS-5: Integration test    | `general-purpose` | `ntm-agentping.test.ts`                          | WS-4       |

### Execution Order (DAG waves)

```
Wave 1: [WS-1]           — bridge module (no deps)
Wave 2: [WS-2, WS-3]    — adapter hooks + HITL gates (parallel, both need WS-1)
Wave 3: [WS-4]           — CLI wiring (needs WS-2 + WS-3)
Wave 4: [WS-5]           — integration test (needs everything)
```

---

## Test Coverage

### Unit Tests (`ntm-agentping.test.ts`)

| Test                        | Given                                | When                            | Then                                              |
| --------------------------- | ------------------------------------ | ------------------------------- | ------------------------------------------------- |
| Bridge sends lifecycle ping | Bridge initialized, daemon healthy   | `onSessionSpawn()` called       | POST to `/api/v1/pings` with notification payload |
| Bridge degrades gracefully  | Daemon offline                       | `onTaskComplete()` called       | No throw, logs warning                            |
| HITL gate blocks execution  | `hitl: true`, approval ping created  | `requestEpicApproval()` called  | Blocks until response or timeout                  |
| Recovery selection works    | Worker error, selection ping created | `requestCrashRecovery()` called | Returns selected action                           |
| Depth encoding correct      | Depth = 2                            | Bridge constructs agentId       | agentId = `lev-ntm-L2-{session}`                  |
| Wave progress pings fire    | Pool executes 3 waves                | Wave lifecycle completes        | 6 pings (start+complete per wave)                 |

### Integration Test

```bash
cd ~/digital/leviathan/core/harness
bun -e "
  import { NTMAgentPingBridge } from './src/providers/ntm-agentping.ts';
  const bridge = new NTMAgentPingBridge({ hitl: false });
  const ok = await bridge.isAvailable();
  console.log('AgentPing available:', ok);
  await bridge.onSessionSpawn({ sessionId: 'test-123', workerCount: 2, workers: [], depth: 0 });
  await bridge.onTaskSent('task-1', 0, 'Write hello world');
  await bridge.onTaskComplete('task-1', 0, { taskId: 'task-1', paneIndex: 0, output: 'done', success: true, promiseDetected: true, durationMs: 5000 });
  await bridge.onSessionShutdown({ totalTasks: 1, successRate: 1.0, totalDurationMs: 5000 });
  console.log('All pings sent — check dashboard at http://localhost:7891');
"
```

### E2E Test (full pipeline)

```bash
lev exec "create a file /tmp/ntm-ap-test.txt with hello" --adapter=ntm --agentping --hitl
# Should: spawn ntm, send lifecycle pings, show on dashboard, create file
```

---

## Success Criteria

1. All 6 ping types appear correctly on AgentPing dashboard (`:7891`)
2. HITL gates block execution until human responds (verified with approval ping)
3. Crash recovery selection returns correct action to pool
4. Bridge degrades gracefully when AgentPing daemon is offline (no crashes, warnings only)
5. Inception depth chain visible in dashboard agentId format
6. Type-check passes with 0 new errors
7. Unit tests: 6+ pass, 0 fail
8. Integration test: pings visible on dashboard

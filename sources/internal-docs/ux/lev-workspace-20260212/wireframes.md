# Wireframes

## Global Nav Map
- Left: Project/entity tree + quick mode switch.
- Top: Active project, transport badge, adapter, cognitive layer.
- Center: Active canvas (chat, board, diff, terminal, heatmap, custom).
- Right: Context rail (approvals, gate status, diagnostics shortcuts).
- Bottom: Voice, stream, and session status tray.

## Expanded Architecture Screen Inventory

1. `WorkspaceShell`
- Purpose: Universal project-first shell and global orientation.
- Primary actions: Switch project, open mode tab, open diagnostics.

2. `ConversationCanvas`
- Purpose: Chat/voice conversation with GenUI rendering.
- Primary actions: Send turn, interrupt, apply action.

3. `ExecutionReview`
- Purpose: Policy/lease checkpoint before privileged execution.
- Primary actions: Approve, deny, inspect rationale.

4. `VoiceControl`
- Purpose: Duplex voice setup and live operation.
- Primary actions: Start/stop, switch providers, barge-in.

5. `SystemStatus`
- Purpose: Daemon/poly/index health overview.
- Primary actions: Refresh, inspect incident, jump to repair.

6. `DiagnosticsAndRepair`
- Purpose: Failure diagnosis and guided repair workflows.
- Primary actions: Inspect cause chain, run repair, escalate.

7. `ConfigInspector`
- Purpose: Resolved config, provenance, safe edits.
- Primary actions: Filter by scope, edit key, validate.

8. `ValidationGateBoard`
- Purpose: Quality gates + architecture passes with evidence.
- Primary actions: Run checks, inspect failures, recheck.

9. `WorkerWaveControl`
- Purpose: Remote worker wave orchestration and recovery.
- Primary actions: Dispatch wave, pause, recover failures.

10. `IndexAndSearchConsole`
- Purpose: Backend-aware search and index freshness controls.
- Primary actions: Query, inspect backend latency, refresh index.

11. `RendererDebugView`
- Purpose: Adapter parity checks and DSL patch diagnostics.
- Primary actions: Switch adapter, replay patch, rollback.

12. `AuditTimeline`
- Purpose: End-to-end trace from chat intent to execution outcome.
- Primary actions: Filter events, inspect provenance, export trace.

## Reduced MVP Screen Set

### Screen 1: WorkspaceShell
Purpose: Launch into one connected project-scoped control surface.
Primary actions: Switch project, select canvas mode, open status.
Layout:
- Header: project selector, adapter badge, transport state.
- Body: left project tree, center active canvas, right context rail.
- Footer: stream/voice/session status.
State variants: idle/loading/empty/error/success.

### Screen 2: ConversationCanvas
Purpose: Run text+voice turns and render GenUI output.
Primary actions: Send prompt, interrupt stream, pin output.
Layout:
- Header: session title + stream state.
- Body: conversation thread + renderer viewport.
- Footer: prompt input + voice toggle.
State variants: idle/loading/empty/error/success.

### Screen 3: ExecutionReview
Purpose: Safely gate privileged execution requests.
Primary actions: Approve, deny, view policy rationale.
Layout:
- Header: action summary + risk level.
- Body: policy/lease details + impact preview.
- Footer: approve/deny controls.
State variants: idle/loading/empty/error/success.

### Screen 4: SystemStatus
Purpose: Keep operational awareness of daemon/poly/index.
Primary actions: Refresh status, inspect degraded service.
Layout:
- Header: global health badge + last updated.
- Body: service health cards + incident timeline.
- Footer: quick jump to diagnostics.
State variants: idle/loading/empty/error/success.

### Screen 5: VoiceControl
Purpose: Configure and run duplex voice interactions.
Primary actions: Toggle mic, switch providers, interrupt output.
Layout:
- Header: voice mode + provider.
- Body: transcript + confidence + playback state.
- Footer: push-to-talk and mute controls.
State variants: idle/loading/empty/error/success.

### Screen 6: ConfigInspector
Purpose: Inspect and edit effective config with provenance.
Primary actions: Filter scope, edit key, validate and save.
Layout:
- Header: scope selector + validation state.
- Body: key/value table with source paths and diffs.
- Footer: save/rollback controls.
State variants: idle/loading/empty/error/success.

## Why These 6 for MVP

- Covers core loop: orient -> converse -> approve execution.
- Covers operational trust: system status + config provenance.
- Covers voice requirement from day one.
- Defers heavy but non-blocking surfaces (workers, full diagnostics, gate board, index console, renderer debug, audit timeline) to Phase 2 while preserving contracts.

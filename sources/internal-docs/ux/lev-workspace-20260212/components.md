# Components

## Expanded Surface Components (Round 2)

1. `WorkspaceShellFrame`
- Purpose: Project-scoped shell with left nav, center canvas, right context rail.
- Props/Data: `activeProject`, `connectionState`, `layoutMode`, `cognitiveLayer`.

2. `ProjectContextSwitcher`
- Purpose: Switch project and preserve/restore context snapshots.
- Props/Data: `projects[]`, `activeProjectId`, `onSwitch`.

3. `ConversationThread`
- Purpose: Render mixed text/tool/ui events from stream.
- Props/Data: `events[]`, `streamState`, `onRetry`, `onInterrupt`.

4. `GenUIRendererHost`
- Purpose: Apply GenUI DSL documents/patches to active surface.
- Props/Data: `document`, `patchQueue`, `adapter`, `theme`.

5. `ExecutionApprovalSheet`
- Purpose: Explicit policy/lease approval for privileged actions.
- Props/Data: `riskSummary`, `policyDecision`, `leaseState`, `onApprove`, `onDeny`.

6. `VoiceDuplexBar`
- Purpose: Drive STT/TTS lifecycle and barge-in behavior.
- Props/Data: `voiceState`, `providers`, `latency`, `confidence`, `onToggle`.

7. `SystemHealthBoard`
- Purpose: Show daemon/poly/index status with degradation context.
- Props/Data: `health`, `incidents`, `lastUpdated`.

8. `DiagnosticsPanel`
- Purpose: Drill into service errors and dependency chain.
- Props/Data: `incident`, `logs`, `dependencies`, `actions`.

9. `RepairActionRunner`
- Purpose: Execute and track guided repair actions.
- Props/Data: `target`, `repairOptions`, `dryRun`, `result`.

10. `ConfigScopeInspector`
- Purpose: Show merged config by scope with provenance.
- Props/Data: `keys[]`, `scope`, `sourcePath`, `effectiveValue`.

11. `ConfigSafeEditor`
- Purpose: Edit config keys with scope and schema guardrails.
- Props/Data: `editableKeys[]`, `schemaErrors`, `onSave`, `onRestore`.

12. `GateScoreboard`
- Purpose: Display quality gates and architecture passes with evidence.
- Props/Data: `gateResults[]`, `severity`, `evidenceRefs`.

13. `WorkerWaveTimeline`
- Purpose: Track worker-wave progress and failure points.
- Props/Data: `wave`, `workers[]`, `approvals`, `events`.

14. `WorkerRecoveryModal`
- Purpose: Reassign/retry/cancel failed workers safely.
- Props/Data: `failedWorkers[]`, `recoveryOptions`, `onApply`.

15. `IndexSearchConsole`
- Purpose: Query search stack with backend visibility.
- Props/Data: `query`, `backends[]`, `latencies`, `results`.

16. `TransportStatusStrip`
- Purpose: Show daemon/poly transport health and endpoint details.
- Props/Data: `protocol`, `endpoint`, `state`, `lastError`, `retry`.

17. `RendererAdapterSwitcher`
- Purpose: Swap runtime renderer adapter (Thesys now, AgentPing later).
- Props/Data: `availableAdapters[]`, `activeAdapter`, `compatibility`.

18. `PatchParityViewer`
- Purpose: Side-by-side adapter parity and patch diff view.
- Props/Data: `sceneId`, `baselineAdapter`, `currentAdapter`, `diff`.

19. `AuditTimelinePanel`
- Purpose: Full trace continuity across chat, policy, and execution.
- Props/Data: `events[]`, `filters`, `exportAction`.

20. `CanvasModeTabs`
- Purpose: Switch among chat/diff/board/terminal/heatmap and custom canvases.
- Props/Data: `tabs[]`, `activeTab`, `onSelect`, `onClose`, `dirtyState`.

## Reduced MVP Component Set

- `WorkspaceShellFrame`
- `ProjectContextSwitcher`
- `ConversationThread`
- `GenUIRendererHost`
- `ExecutionApprovalSheet`
- `VoiceDuplexBar`
- `SystemHealthBoard`
- `TransportStatusStrip`
- `ConfigScopeInspector`
- `ConfigSafeEditor`

## Screen to Component Mapping (MVP)

- `WorkspaceShell`
  - `WorkspaceShellFrame`, `ProjectContextSwitcher`, `CanvasModeTabs`, `TransportStatusStrip`

- `ConversationCanvas`
  - `ConversationThread`, `GenUIRendererHost`, `VoiceDuplexBar`

- `ExecutionReview`
  - `ExecutionApprovalSheet`, `AuditTimelinePanel`

- `SystemStatus`
  - `SystemHealthBoard`, `TransportStatusStrip`

- `VoiceControl`
  - `VoiceDuplexBar`, `ConversationThread`

- `ConfigInspector`
  - `ConfigScopeInspector`, `ConfigSafeEditor`

## Composition Rules

- Keep `GenUIRendererHost` pure and contract-driven; adapters implement rendering strategy only.
- Keep `ConversationThread` decoupled from execution internals; execution appears as stream events.
- Keep approvals explicit and visible; no hidden mutation side effects.
- Keep diagnostics/repair separated from config editing to preserve bounded contexts.

# PRD 4: AgentPing Human Loop (LEV Integration Contract)

**Status**: Active consolidation (2026-02-11)

This document defines what belongs in AgentPing vs LEV for the human-loop action surface architecture.

## Product Intent

AgentPing is the interaction surface layer for agent actions:

- capture intent from voice/text/events
- render actionable UI surfaces for review/approval/progress
- return structured human decisions to execution systems

LEV remains the runtime orchestration and execution authority.

## System Boundary (Authoritative Split)

| System | Owns | Does not own |
|---|---|---|
| LEV | orchestration runtime, lifecycle/state graph, policy evaluation, worker execution, provenance | reusable UI component canon |
| AgentPing | human-loop protocol, component kit, canvas/studio, dashboard control plane, approval/review UX | long-running worker execution internals |
| AgentLease (POC) | lease issuance, step-up auth (2FA/voice/passphrase), scoped permission grants | renderer ownership and generic orchestration |

## Current State

- Runner-managed surfaces are live in `community/agentping/packages/dashboard-runner/config/dashboards.yaml`.
- UI ownership is fragmented across `packages/ui`, `packages/adapters/web-ui`, `packages/studio`, and `packages/canvas`.
- Canvas payload contract is strict (`sofia-widget` envelope), but canvas render ownership is not yet fully sourced from `packages/ui`.
- Studio has shell/runtime responsibilities plus overlapping canvas/orchestration behavior.
- Voice-first mode exists as design direction and partial UI, not as fully integrated runtime flow.

## Ideal State

1. **Single UI kit** in `community/agentping/packages/ui`.
2. **Single canvas runtime** with modes (`render`, `inspect`, `compose`) consumed by Studio and web surfaces.
3. **Adapter-only adapter layer** where `packages/adapters/*` owns transport/integration concerns only.
4. **Single dashboard control plane** (`dashboard-runner` + manager server), with Studio as a client.
5. **Portable install model** so external clients can install AgentPing surfaces, add adapters, and theme safely.

## Install + Extension Model (Target)

Client outcomes:

- install AgentPing core + UI/canvas packages
- run dashboard runner + manager UI + studio shell
- register custom adapter(s) for their action surfaces (web/mobile/extension/desktop)
- register custom themes through explicit theme registry (fail-fast on unknown theme)

## Security Execution Model (North Star)

Protected action flow:

1. Agent proposes action through AgentPing surface.
2. AgentLease performs step-up auth and issues lease.
3. LEV evaluates ABAC policy and creates ephemeral sandbox worker (docker/browser-capable).
4. Worker executes scoped action and streams progress.
5. AgentPing renders checkpoints and captures approvals.
6. Worker is destroyed at completion; provenance remains.

## UX Operating Modes (Target)

- **Desktop wide**: multi-panel command center (portfolio + urgent queue + live canvases).
- **Tablet/mobile**: progressive disclosure, voice-first mode with visual checkpoints.
- **Voice-only fallback**: no hidden automation; explicit review points for protected actions.

## What Must Move Out Of AgentPing

- execution-runtime concerns that belong to LEV (worker lifecycle, deep policy engine)
- overloaded adapter responsibilities in `packages/adapters/web-ui` that are actually reusable UI canon

## What Must Stay In AgentPing

- human-loop protocol and payload contracts
- reusable interaction primitives and component kit
- dashboard runner/operator UI and canvas/studio interaction surfaces

## Current Critical Gaps

- Shared component canon is not fully centralized in `packages/ui`.
- Canvas render paths are not yet fully unified to UI-kit exports.
- Theme strictness is inconsistent across some legacy surfaces.
- Protected-action lease gate is a target model, not full production path yet.

## Immediate Execution Order

1. Consolidate shared components into `packages/ui`.
2. Unify canvas runtime and remove duplicated render logic.
3. Reduce `packages/adapters/web-ui` to adapter-shell concerns only.
4. Remove secondary Studio orchestration path after parity validation.
5. Integrate AgentLease gate with explicit protected-action checkpoints.
6. Wire LEV ephemeral worker path behind lease + ABAC enforcement.

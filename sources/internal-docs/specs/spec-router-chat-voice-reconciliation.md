---
module: "router"
title: "FlowMind + Stream Router Chat/Voice Reconciliation"
type: spec
entity: "spec-router-chat-voice-reconciliation"
stage: crystallized
created: "2026-02-13"
author: "codex"
status: draft
---

# FlowMind + Stream Router Chat/Voice Reconciliation — Behavioral Specification

## Executive Summary
This spec defines the planning-to-execution contract for chat/voice reconciliation between FlowMind routing and stream-router schema. It standardizes boundaries, validation gates, and BD task decomposition without prescribing implementation internals.

## Context

This spec aligns with canonical C1-C4 architecture and parser-at-tip prior art:
- `docs/01-architecture.md`
- `docs/specs/spec-router-flowmind-parser.md`
- `context/schemas/stream-router.yaml`
- `context/schemas/flowmind-types.yaml`
- `core/flowmind/src/router.ts`

### Existing State
- `conversation` and `voice` are present in router and schema contracts.
- Stream-router now uses canonical handler IDs for chat/voice routing.
- Sidequest is treated as an app-level route and should remain outside low-level SDK/CLI primitive ownership.
- Parser-at-tip prior art identifies a gap between `chat.conversation` emission and concrete subscriber-driven flow execution.
- Execution surfaces are in transition: legacy wrapper paths still carry orchestration logic (`core/harness/src/commands/exec.ts`) while target ownership is SDK-first + poly-driven pass-through.

### Target State
- A single contract set defines chat/voice ingress behavior, lifecycle states, and route ownership.
- Spec passes PROPOSE -> SPEC and SPEC -> EXECUTE gate preconditions, including BDD scenarios, BD task decomposition, and rollback plan.
- Sidequest boundary remains explicit and testable as an architectural invariant.
- Runtime move is SDK-first pass-through behind a FlowMind kernel tip flag: keep current parser behavior, gate through `@lev-os/exec` first, and progressively route wrappers to SDK execution while preserving 1:1 behavior.

## User Scenarios (BDD)

### Scenario 1: Conversation ingress is classified and routed consistently
```gherkin
Feature: Conversation ingress reconciliation
  As a planning team
  I want conversation input to resolve to a single declared routing contract
  So that schema, router, and lifecycle artifacts do not drift

  Scenario: Conversation message enters routing surface
    Given a message classified as conversation
    When routing policy is applied
    Then the message is assigned to captured lifecycle state
    And the route uses the canonical conversation handler contract
```

### Scenario 2: Voice ingress follows the same contract discipline
```gherkin
Feature: Voice ingress reconciliation
  As a planning team
  I want voice input to follow the same boundary and lifecycle rules as conversation
  So that chat/voice behavior is coherent and auditable

  Scenario: Voice transcript enters routing surface
    Given a payload classified as voice
    When routing policy is applied
    Then the payload is assigned to captured lifecycle state
    And the route uses the canonical voice handler contract
```

### Scenario 3: Sidequest stays application-level only
```gherkin
Feature: Sidequest boundary guard
  As an architecture owner
  I want sidequest excluded from low-level primitive contracts
  So that SDK/CLI/router layers remain stable and focused

  Scenario: Low-level contract audit runs
    Given low-level router/schema/CLI surfaces
    When boundary checks are evaluated
    Then sidequest is absent from low-level primitive ownership
    And sidequest remains available only as an app-level skill/subagent route
```

## Behavioral Specification

### Inputs
- Planning signals from architecture and prior-art documents.
- Routing contracts from stream-router and flowmind-types schemas.
- Router lifecycle/lane mapping for `conversation` and `voice`.
- Boundary policy: sidequest is app-level only.

### Processing
1. Reconcile C1-C4 context against current router/schema surfaces.
2. Validate lifecycle and routing consistency for chat/voice types.
3. Validate boundary constraints for sidequest ownership.
4. Generate execution-ready planning outputs: contracts, tasks, and test expectations.

### Outputs
- One reconciled behavioral spec (this document).
- BD task decomposition aligned to scenarios.
- Validation gate checklist coverage suitable for SPEC -> EXECUTE.

### Performance
- Contract-level reconciliation should complete within one planning cycle.
- Validation checks must remain deterministic and repeatable.

## Contract

### Dependencies
- Core docs/specs listed in Context.
- Work skill template/gate contracts:
  - `.agents/skills/work/templates/spec.md`
  - `.agents/skills/work/references/gates.md`

### Integration Points
- `context/schemas/stream-router.yaml` (content types + routes)
- `context/schemas/flowmind-types.yaml` (artifact typing)
- `core/flowmind/src/router.ts` (classification/lane/lifecycle mapping)
- `core/flowmind/src/router.test.ts` (contract verification)

### Breaking Changes
- None required by this spec itself.
- Any future runtime or handler ownership shift must be captured as explicit spec revision, not implicit drift.

## Implementation Guidance

### Recommended Skills
- `work`
- `lev-cdo` (if deeper deliberation is required)
- `lev-research` (if additional evidence is needed)

### Team Structure
- Planning mode with parallel planners.
- One consolidator agent owns final spec normalization and gate checks.

### Workstreams
- `WS1` Contract Surface: `lev-4uygq.1` -> `lev-4uygq.2` + `lev-4uygq.3` (chat contract + SDK/daemon surfaces).
- `WS2` Secure Exec Boundary + Spawn Semantics: `lev-4uygq.8` + `lev-4uygq.10`.
- `WS2b` SDK Pass-Through Move: `lev-4uygq.13` + `lev-4uygq.14`.
- `WS3` Conversation/Voice Adapters + UX Surfaces: `lev-4uygq.4` + `lev-4uygq.5` + `lev-4uygq.6` + `lev-4uygq.7` + `lev-4uygq.9`.
- `WS4` Validation/Telemetry: `lev-4uygq.8.1` + `lev-4uygq.8.2` + `lev-4uygq.11`.

## BD Task Decomposition

### Epic Mapping

| Epic ID | Title | Scope | Success Signal |
|---------|-------|-------|----------------|
| lev-4uygq | Lev Chat Plane (SDK/Daemon) + Voice Surface | Conversation plane contract + SDK/daemon gateway + adapters + app/voice surfaces | Scenario 1/2/3 pass in chat/voice integration and security boundary tests |

### Task Breakdown

| Task ID | Summary | Linked Scenario(s) | Owner | Acceptance Signal |
|---------|---------|--------------------|-------|-------------------|
| lev-4uygq.1 | Chat contract and event schema separation (conversation vs execution plane) | Scenario 1, Scenario 3 | Contracts owner | Contract fixtures and docs pass with no overlap and no hidden fallback path |
| lev-4uygq.2 | SDK chat surface (session + stream lifecycle) | Scenario 1 | SDK owner | Deterministic stream lifecycle coverage (happy path/cancel/reconnect) |
| lev-4uygq.3 | Daemon chat gateway (RPC/HTTP streaming) | Scenario 1, Scenario 2 | Daemon owner | Gateway start/stop/stream multiplexing + disconnect behavior validated |
| lev-4uygq.10 | Spawn semantics for local/remote worker-as-tool profiles | Scenario 2, Scenario 3 | Coordination owner | Local/remote spawn-wait parity and policy-bounded execution proven |
| lev-4uygq.8 | Execution security boundary (deny-by-default, anti-injection, non-destructive defaults) | Scenario 3 | Security owner | Privileged execution blocked without explicit policy/lease checks |
| lev-4uygq.13 | SDK-first pass-through (`harness/cli/exec` thin wrappers over `@lev-os/exec`) | Scenario 1, Scenario 2, Scenario 3 | SDK owner | FlowMind kernel tip flag controls rollout, `chat/worker.harness.flow.yaml` overlays are validated, and 1:1 behavior parity is preserved |
| lev-4uygq.14 | Poly surface generation for CLI/MCP/daemon access | Scenario 1, Scenario 2, Scenario 3 | Poly owner | CLI/MCP/daemon access resolves from poly declarations with deterministic generation artifacts |
| lev-4uygq.8.1 | E2E gate matrix (live usage, monotonic aggregation, trace replay) | Scenario 1, Scenario 2, Scenario 3 | Validation owner | Gate report shows passing token + aggregation + trace assertions |
| lev-4uygq.8.2 | TDD-triad validation gates for pass-through refactor | Scenario 1, Scenario 2, Scenario 3 | Validation owner | RED/GREEN/REFACTOR evidence + parity/perf/thin-wrapper gates enforced |
| lev-4uygq.11 | Usage telemetry ownership migration to plugins/platforms | Scenario 1, Scenario 2 | Platform owner | Provider usage provenance contract documented and validated |

### Dependency Ordering

| Blocking Task | Dependent Task | Reason |
|---------------|----------------|--------|
| lev-4uygq.1 | lev-4uygq.2 | SDK surface must implement locked conversation-plane contract |
| lev-4uygq.1 | lev-4uygq.3 | Daemon gateway schema must match locked contract/event taxonomy |
| lev-4uygq.3 | lev-4uygq.10 | Spawn/wait semantics require canonical gateway and transport behavior |
| lev-4uygq.1 | lev-4uygq.13 | Pass-through wrappers must preserve contract/event taxonomy |
| lev-4uygq.2 | lev-4uygq.13 | SDK chat surface is prerequisite for SDK-first wrapper delegation |
| lev-4uygq.3 | lev-4uygq.13 | Gateway behavior must remain equivalent across CLI/SDK paths |
| lev-4uygq.13 | lev-4uygq.14 | Poly generation layer follows pass-through execution contract |
| lev-4uygq.2 | lev-4uygq.8.1 | Live usage gate depends on SDK-emitted normalized usage fields |
| lev-4uygq.3 | lev-4uygq.8.1 | Trace replay and stream assertions depend on daemon event flow |
| lev-4uygq.13 | lev-4uygq.8.2 | TDD-triad gates evaluate pass-through behavior and thin-wrapper constraints |
| lev-4uygq.14 | lev-4uygq.8.2 | Gate set must validate generated CLI/MCP/daemon surfaces from poly declarations |

## Test Coverage

### Unit Tests
- Router classification tests include `conversation` and `voice`.
- Lifecycle mapping tests assert captured-state behavior.

### Integration Tests
- Schema/contract consistency checks between stream-router and router expectations.
- Boundary checks confirm sidequest absence from low-level ownership surfaces.

### E2E Verification
- `node tooling/scripts/check-cli-surface.js`
- YAML parse checks for:
  - `context/schemas/flowmind-types.yaml`
  - `context/schemas/stream-router.yaml`
- `cd core/flowmind && pnpm -s test src/router.test.ts`

## Success Criteria
- BDD scenarios are complete and testable.
- Contract dependencies/integration points are explicitly documented.
- BD task decomposition exists and maps directly to scenarios.
- Rollback plan is present and actionable.
- Spec is ready for SPEC -> EXECUTE gate review.

## Rollback Plan

### Rollback Trigger
- Spec gate failure caused by unresolved contract conflicts or boundary violations.

### Rollback Steps
1. Revert to previous approved proposal/spec baseline for this slice.
2. Isolate failing contract area and mark it as blocked in task state.
3. Re-run planning alignment and produce a revised spec patch.

### Data/State Recovery
- Restore previous validated schema/router contract references from VCS.
- Preserve audit evidence and failed gate diagnostics for comparison.

## Open Questions
- Should final runtime ownership for chat/voice handler execution be centered in `core/triggers` or `core/flowmind`?
- What is the canonical source for route precedence when schema action labels and router lanes diverge?
- Do we require an explicit policy test fixture for the sidequest boundary in CI gate checks?
- Should we enforce wrapper thinness by static threshold (LOC/function complexity) or by architectural lint rules tied to import boundaries?

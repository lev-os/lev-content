# Summary

[91% confident] This second-pass UX expands beyond chat/canvas basics to include the full control-plane architecture from `docs/design` and `docs/specs` (daemon, poly, index, config provenance, validation gates, repair, worker waves), then intentionally reduces to a 6-screen MVP that preserves architectural integrity.

## Problem + Success Criteria
- Problem: Lev has strong runtime primitives but fragmented operator surfaces; users need one project-first, voice-capable workspace with clear chat/exec boundaries and observable system health.
- Success: The UX model now includes both a complete architecture inventory and a deliberate MVP subset, with acceptance checks tied to `.lev/validation-gates.yaml`.

## Key Tradeoffs
- We prioritize architectural correctness and operational transparency over a narrower feature-only MVP.
- We keep renderer adapters swappable and dumb, trading early implementation speed for long-term portability.
- We defer advanced operational surfaces to Phase 2 while maintaining data contracts in MVP.

## Expanded Coverage Added in Round 2
- System health across daemon/poly/index.
- Diagnostics + repair workflow.
- Config scope/provenance editing.
- Validation gate and architecture pass visibility.
- Worker-wave orchestration and recovery.
- Backend-aware index/search console.
- Adapter parity and audit timeline surfaces.

## Reduced MVP (Implemented First)
1. WorkspaceShell
2. ConversationCanvas
3. ExecutionReview
4. SystemStatus
5. VoiceControl
6. ConfigInspector

## Acceptance Alignment
- Interaction models now include measurable acceptance checks per screen.
- Validation profile references mandatory checks, quality gates, and architecture passes from `.lev/validation-gates.yaml`.
- MVP retains explicit policy gating and fail-fast behavior.

## Open Questions
- Which transport becomes first-class for v1 streaming UX: WebSocket-first or HTTP-first with WS upgrade?
- Should `SystemStatus` include lightweight incident repair actions in MVP, or keep repair strictly in Phase 2?
- Which local voice provider is default in installer path for first release?

## Smallest Shippable Slice
- Build screens 1-6 using the defined component set and FSM acceptance checks.
- Keep expanded surfaces represented as routes/contracts, but hidden behind feature flags until Phase 2.
- Validate with gate-aware smoke tests before spec promotion.

→ Next: Promote this UX run into implementation-ready specs with one BD epic per MVP screen and gate-linked DoD.
→ Related: docs/design/contracts.md, docs/design/core-index.md, docs/design/core-other.md, docs/specs/spec-router-flowmind-parser.md, docs/specs/spec-unify-agentping-surfaces.md

💡 Tip: If you preserve entity contracts for deferred screens now, Phase 2 becomes additive routing work instead of a refactor.

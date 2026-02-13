---
module: "graph"
design_refs:
  - "docs/design/graph-primitives.md"
  - "docs/design/dimension-model.md"
code_refs:
  - "src/core/graph/**"
validation_gates:
  - "gate:graph-consistency"
---


**Status:** CANONICAL SPEC (promoted from inbox 2026-02-12)
**Score:** 70/100 (design complete, implementation incomplete)
**Date:** 2026-02-04 (original review), 2026-02-12 (promoted)
**Source:** CDO multi-agent review of entity lifecycle plan → inbox processing session
**Epic:** lev-cpnv (Lifecycle FSM Consolidation)
**Tier:** T1/T2 (graph storage = T1 Kernel, proposal FSM = T2 Infrastructure)
**Depends on:** `docs/design/graph-primitives.md`, `docs/design/vernacular.md`, `docs/design/actor-graph.md`

---

## Executive Summary

**Graph-first entity model**: Architecturally sound
**SDLC as plugin**: Correct approach
**Implementation plan**: Incomplete and risky

The conceptual model (Entity/Claim/Evidence/Proposal/View) is excellent. The execution plan needs work before implementation.

---

## Strengths

### Graph-First Architecture
- **Separation of concerns**: Truth (Claims) vs Interaction (Proposals) vs Presentation (Views)
- **Explicit provenance**: Every entity tracks source and via
- **Link-based dependencies**: Much better than implicit coupling
- **State machine foundation**: LifecycleState enum supports proper FSM

### SDLC as Plugin
- Graph primitives (Layer 0) are domain-agnostic
- SDLC workflows (Layer 1) compose graph operations
- BD integration maps naturally to graph mutations
- Skills like `/interview`, `/plan` become graph operation composers

### BD as Graph Driver
- `bd create` → `add_entity` operation
- `bd update` → `patch_entity` operation
- Task dependencies → `blocked_by` links
- Eliminates impedance mismatch between tracker and system

---

## Critical TODO Items

### TODO 1: Type Migration Path

**Problem**: 4 existing event types with no migration strategy to LevEvent.

**Current fragmentation**:
- `DomainEvent` in core/events/src/events/base.ts
- `LifecycleEvent` in core/agent-adapter (hooks)
- `GateEvent` in core/lifecycle (validation)
- `CloudEvents` in bridges

**Required**:
```typescript
// PHASE 1: Adapters (coexistence)
class DomainEventAdapter implements LevEvent { ... }
class LifecycleEventAdapter implements LevEvent { ... }

// PHASE 2: Deprecation warnings (6 months)
// PHASE 3: Runtime errors (1 year)
// PHASE 4: Remove old types (18 months)
```

**Deliverables**:
- [ ] Adapter classes for each legacy event type
- [ ] Deprecation timeline with dates
- [ ] Feature flag for new event path
- [ ] Migration guide for consumers

---

### TODO 2: GraphOperation Transaction Model

**Problem**: No atomicity semantics for operations.

**Failure case**:
```typescript
operations: [
  { op: 'add_entity', entity: spec },  // succeeds
  { op: 'add_link', from: 'spec', to: 'nonexistent' },  // fails
  { op: 'patch_entity', entity_id: 'spec', patch: [...] }  // orphaned
]
```

**Required**:
```typescript
interface ProposalExecution {
  proposal: Proposal;
  execution_mode: 'atomic' | 'best_effort' | 'compensating';
  rollback_strategy?: CompensatingOperation[];
}
```

**Deliverables**:
- [ ] Define execution mode semantics
- [ ] Implement rollback for atomic mode
- [ ] Define conflict resolution (last-write-wins vs CRDT)
- [ ] Add transaction ID to operations

---

### TODO 3: View Rendering Pipeline

**Problem**: Views defined but no rendering semantics.

**Unanswered questions**:
1. **Query language**: GraphQL? Datalog? Custom DSL?
2. **Freshness**: Event-based refresh or polling?
3. **Caching**: Where? Invalidation strategy?
4. **Rendering**: Template-based or programmatic?

**Required**:
```typescript
interface ViewRenderer {
  query(view: View): GraphQueryResult;
  transform(result: GraphQueryResult, layout: ViewLayout): RenderTree;
  render(tree: RenderTree, format: 'markdown' | 'html'): string;
}
```

**Deliverables**:
- [ ] Define query language (recommend: simple path expressions)
- [ ] Implement view query executor
- [ ] Define template system (recommend: Mustache)
- [ ] Add cache layer with event-driven invalidation

---

### TODO 4: Proposal Lifecycle FSM

**Problem**: Proposals have status but no lifecycle hooks.

**Required FSM**:
```
pending → [gates_check] → passed/failed
  → failed → amended → pending (retry)
  → passed → accepted → [apply_operations]
  → rejected (user override)
```

**Deliverables**:
- [ ] Define Proposal FSM in XState
- [ ] Add lifecycle hooks (onPending, onAmended, onAccepted, onRejected)
- [ ] Wire gates_check to ValidationGateExecutor
- [ ] Emit LevEvents on state transitions

---

### TODO 5: Gate Execution Interface

**Problem**: Gate taxonomy is names only, no implementation spec.

**Required**:
```typescript
interface Gate {
  id: string;
  name: string;
  severity: 'CATASTROPHIC' | 'CRITICAL' | 'WARNING';
  check: (proposal: Proposal, context: GateContext) => Promise<GateResult>;
  claim_template: Claim;  // Emits claim on pass/fail
}

interface GateResult {
  passed: boolean;
  message: string;
  evidence?: Evidence;
  claim_emitted?: Claim;
}
```

**Gate taxonomy v0**:
| Gate | Severity | Check |
|------|----------|-------|
| Gate/Scope | CRITICAL | All changes within project boundary |
| Gate/Safety | CATASTROPHIC | No secrets exposed |
| Gate/Feasibility | CRITICAL | Tools available, permissions granted |
| Gate/Standards | WARNING | Formatting, naming conventions |
| Gate/Progress | WARNING | Measurable advancement |

**Deliverables**:
- [ ] Define Gate interface
- [ ] Implement 5 core gates
- [ ] Add gate composition (AND/OR)
- [ ] Wire to Proposal lifecycle

---

### TODO 6: Graph Storage Backend

**Problem**: Nowhere specified where the graph lives.

**Options**:
| Option | Pros | Cons |
|--------|------|------|
| In-memory + JSON file | Simple, fast | No query, single process |
| SQLite | Embedded, queryable | Schema migrations |
| Postgres | Scalable, ACID | Deployment complexity |
| Event sourcing | Audit trail, replay | Complexity |

**Recommendation**: Start with in-memory + JSON file, migrate to event sourcing.

**Deliverables**:
- [ ] Define IGraphStore interface
- [ ] Implement JsonFileGraphStore (v0)
- [ ] Add event log for mutations
- [ ] Plan migration to EventSourcedGraphStore

---

### TODO 7: Event Ordering and Replay

**Problem**: LevEvent has correlationId/causationId but no replay semantics.

**Required**:
- Event store with append-only log
- Event versioning for schema evolution
- Replay logic: `graph_state = reduce(events)`
- Snapshotting for performance

**Deliverables**:
- [ ] Define EventStore interface
- [ ] Implement JsonlEventStore (builds on existing)
- [ ] Add replay() method
- [ ] Add snapshot support

---

## Contradictions to Resolve

### C1: Documents as Views vs Documents as Entities

**Part 3** says:
> ##-thing.md files are Views, not truth sources

**But spec template** treats spec.md as the spec entity.

**Resolution needed**: Pick one:
- Specs are Views → need a way to edit underlying entity
- Specs are Entities → drift from markdown possible

**Recommendation**: Specs are Views. Editing View updates underlying Entity via Proposal.

---

### C2: Claims as Assertions vs Claims as Tests

**Claim definition**:
```typescript
predicate: "is_approved"  // Assertion about state
```

**Spec template uses**:
```markdown
predicate: "WHEN user clicks login THEN system SHALL authenticate"  // Test case
```

**These are different**:
- Assertion: "The system is approved"
- Test case: Runnable specification

**Resolution**: Tests generate Evidence for Claims. A test passing creates `evidence/test-run-xyz` which supports `claim/ac-1`.

---

### C3: BD Status vs LifecycleState Mapping

**BD statuses**: open | closed | in_progress | blocked | tombstone

**LifecycleState**: ephemeral | captured | classified | crystallizing | crystallized | manifesting | completed | discarded

**Missing**: Explicit mapping.

**Proposed mapping**:
| BD Status | LifecycleState | Notes |
|-----------|----------------|-------|
| open | captured or crystallized | Depends on DoR |
| in_progress | manifesting | Active work |
| blocked | crystallizing | Needs unblock |
| closed | completed | Done |
| tombstone | discarded | Rejected |

---

## Implementation Order

### Phase 0: Design Completion (2-3 days)
- [ ] Answer all TODOs above
- [ ] Resolve contradictions
- [ ] Write interface specifications

### Phase 1: Minimal Graph (1 week)
- [ ] Entity + Link only (no Claims/Proposals)
- [ ] JsonFileGraphStore
- [ ] Single LevEvent type (wrap existing)

### Phase 2: One SDLC Workflow (1 week)
- [ ] `/plan` → add_entity(spec) → render view
- [ ] Proves the layering works
- [ ] Identifies integration pain points

### Phase 3: Claims + Evidence (1 week)
- [ ] Add Claim, Evidence types
- [ ] Implement Gate/Safety
- [ ] Test with one validation rule

### Phase 4: Proposals (2 weeks)
- [ ] Proposal type + FSM
- [ ] Transaction model
- [ ] Approval workflow

### Phase 5: Full SDLC Plugin (ongoing)
- [ ] All slash commands as graph composers
- [ ] BD → Graph bridge
- [ ] View rendering for all entity types

---

## Testing Strategy

### Unit Tests
- [ ] GraphOperation application
- [ ] LifecycleState transitions
- [ ] Gate evaluation

### Integration Tests
- [ ] Proposal → Gates → Apply → View
- [ ] BD command → GraphOperation → Event

### Property-Based Tests
- [ ] FSM invariants: `e.state = 'manifesting' ⟹ exists task t: t.links(implements, e)`
- [ ] Graph consistency: No orphan links

---

## Rollout Strategy

1. **Feature flag**: `GRAPH_FOUNDATION_ENABLED`
2. **Parallel run**: Old + new systems for 2 weeks
3. **Canary**: Enable for one project first
4. **Full rollout**: After validation

---

## References

- Plan file: `/Users/jean-patricksmith/.claude-sneakpeek/claudesp/config/plans/keen-wibbling-platypus.md`
- Truth docs: `~/clawd/.lev/pm/truth/07-core-lifecycle.md`
- BD Epic: lev-cpnv

---

**Next action**: Address TODO 1-7 before writing code.

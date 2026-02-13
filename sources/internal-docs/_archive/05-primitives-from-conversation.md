# Primitives (From Gold Mine Conversation)

## Core Primitives

### View
Renderable projection over graph state. NOT truth. Always scoped.

### Claim
Atomic assertion about world/graph. Can be true/false/unknown/disputed.

### Evidence
Support for a claim (logs, tests, measurements, user attestation).

### TruthState
System's stance on a claim under scope + time:
- accepted
- rejected
- unknown
- disputed
- stale

### Proposal
Candidate mutation of graph (add/change/remove claims, entities, links).

### EphemeralReasoning
Non-canonical thinking that generates proposals. Don't promote unless gated.

## Key Insight

**Document is NOT truth. Document is View over graph.**

"Truth" is computed from Claims + Evidence + TruthState, not stored in files.

## Schema Sketch

```yaml
claim:
  id: "claim/port-collisions-zero"
  subject: "project/leviathan"
  predicate: "has_constraint"
  object: { kind: "Constraint", value: "Zero port collisions" }
  scope: { workspace: "local-dev", time_range: "now" }
  truth_state: "unknown"
  evidence_refs: []

evidence:
  id: "evidence/daemon-scan-001"
  kind: "observation"
  source: "daemon/ports"
  payload_ref: "artifact/logs/ports.json"
  collected_at: "2026-02-04T10:20:00Z"
  reliability: 0.8

view:
  id: "view/project-overview"
  target: "project/leviathan"
  type: "ProjectOverview"
  query: { include: [...], depth: 2 }
  layout: { mode: "sidebar-panels" }
```

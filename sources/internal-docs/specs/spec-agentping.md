---
module: "agentping"
design_refs:
  - "docs/design/vernacular.md"
  - "docs/design/contracts.md"
code_refs:
  - "src/core/agentping/**"
validation_gates:
  - "gate:agentping-protocol-conformance"
---

# AgentPing Specification (Core)

This specification unifies the AgentPing protocol, surface unification, and remote worker integration.

## Design Refs
- `spec-agentping-remote.md` (Extension)
- `spec-agentping-surfaces.md` (Extension)

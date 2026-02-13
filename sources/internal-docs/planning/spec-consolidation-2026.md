# Proposal: Spec Consolidation & Base Dev Harness 🏗️

**Status:** PROPOSAL
**Author:** @jean-patricksmith
**Date:** 2026-02-13

---

## 1. Executive Summary

Current state: 36+ fragmented specs, heavy overlap in "Router" definitions, and no canonical definition for execution harness configuration.
Proposal: 
1. **Consolidate** fragmented Router specs into a unified `spec-flowmind-core.md`.
2. **Formalize** the "Base Dev Harness" with a configuration-driven schema (FlowMind YAML) that exposes "knobs" for edit capabilities (Hashline vs Patch vs Replace).
3. **Map** the relationships between FlowMind (Policy), Exec (SDK), and Harness (Runtime).

---

## 2. Spec Inventory & Relationships

### 2.1 The Current Fragmented Map
- **Router Cluster:** `spec-router.md`, `spec-router-flowmind-parser.md`, `spec-router-flowmind-ux.md`, `spec-router-chat-voice-reconciliation.md`. (4 files, high overlap).
- **Execution Cluster:** `core-sdk.md` (design), `core-harness` (code implied), `spec-agentfs.md`.
- **Lifecycle Cluster:** `spec-lifecycle.md`, `spec-lifecycle-hooks.md`.

### 2.2 The Proposed Unification
We propose collapsing the "Router Cluster" into:
- **`spec-flowmind-core.md`**: The compiler, parser, and policy engine (was `spec-router-*`).
- **`spec-harness-config.md`**: The runtime environment configuration (New).

**Relationship Map:**
```mermaid
graph TD
    FlowMind[FlowMind (Policy Engine)] -->|Compiles to| ExecRequest[ExecRequest (Intent)]
    ExecRequest -->|Dispatched to| ExecSDK[Exec SDK (@lev-os/exec)]
    ExecSDK -->|Routes via| HarnessConfig[Harness Policy (YAML)]
    HarnessConfig -->|Configures| Runtime[Runtime Environment (Chat / Voice / Dev / Worker)]
    Runtime -->|Exposes| Capabilities[Capabilities (Hashline / Patch / Docker)]
```

---

## 3. The Base Dev Harness (Hashline)

We propose a new spec **`spec-harness-config.md`** to define the "Base Dev Harness" as a configurable Policy, not just a TypeScript class.

### 3.1 Harness Config Schema (YAML)
Every harness (Chat vs Dev vs Worker) will be defined by a FlowMind Policy:

```yaml
# core/flowmind/policies/dev.harness.yaml
type: harness
extends: base-worker
name: dev-standard

capabilities:
  # The "Knobs" for Edit Format (The "Hashline" Insight)
  edit_format:
    mode: dynamic # Auto-select based on model
    options:
      - hashline # Anchored edits (Best for low-recall models)
      - patch    # Unified diffs
      - replace  # String replacement
    fallback: replace
  
  safety:
    mode: relaxed # Allows shell execution
    container: false # Runs on host (for now)

  tools:
    - name: read_file
    - name: write_file
    - name: exec_command
    - name: git_status
```

### 3.2 The "Hashline" Capability
The spec will explicitly define "Hashline" as a supported edit capability:
- **Read**: Returns lines with 3-char hash anchors (`1:a3| code...`).
- **Write**: Accepts edits referencing these anchors.
- **Goal**: Eliminate "recall hallucination" errors in weak models.

---

## 4. Work Execution Plan (PROPOSE DO NOT DO)

1.  **Rename/Merge**: 
    -   `spec-router-flowmind-parser.md` + `spec-router.md` -> `spec-flowmind-core.md`.
    -   Archive the old router specs.
2.  **Create**:
    -   `docs/specs/spec-harness-config.md`.
3.  **Update**:
    -   `docs/specs/spec-flowmind-chat-voice.md` to reference `spec-harness-config.md` for policy definitions.

---

## 5. CDO Workflow Alignment for 2026
This structure aligns with the **Chief Design Officer (CDO)** workflow:
- **Design (Specs)**: Configuration-first (YAML).
- **Policy**: Defined in FlowMind.
- **Execution**: Handled by Exec SDK (dumb pipes, smart config).

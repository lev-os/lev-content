# Spec: FlowMind Chat/Voice Reconciliation + Validation Wiring

**Status:** DRAFT
**Owner:** @jean-patricksmith
**Tier:** T2 (Standard)
**Design Refs:** 
- `docs/design/core-flowmind.md`
- `docs/design/core-sdk.md`
**Code Refs:**
- `core/exec/**` (Runtime)
- `core/flowmind/src/router.policy.ts` (Policy)
- `core/validation/src/**` (Validation Framework)

---

## 1. Executive Summary

This spec reconciles the "Router" concept (now a **Policy**) with the "Exec" concept (now a **Module**), validates the separation of concerns, and defines the wiring for **Project + Global Validation Gates**.

The `Exec` module (`core/exec`) is the universal entry point. The `FlowMind` router (`core/flowmind`) is a policy engine invoked *by* Exec. Validation Gates are loaded via a standard FlowMind workflow and applied during the `gate` phase of execution.

---

## 2. Architecture & Boundaries

### 2.1 Router vs. Exec
- **`core/exec` (Module):** The runtime mechanism. It receives `ExecRequest`, runs the `FlowMindGate`, and dispatches to adapters (Claude, Shell, SDK).
- **`core/flowmind` (Policy):** The decision engine. Contains `router.policy.ts` (8-factor analysis) and `schema.ts`. It does *not* execute code directly; it *compiles* intent into usage of `exec`.

### 2.2 Validation Wiring
- **UniversalValidationSystem:** The code framework in `core/validation`.
- **Gate Loader:** A **FlowMind Workflow** that:
  1. Loads `~/.lev/validation-gates.yaml` (Global).
  2. Loads `.lev/validation-gates.yaml` (Project).
  3. Merges them.
  4. Configures the `UniversalValidationSystem`.

---

## 3. Data Compatibility (Contracts)

### 3.1 ExecRequest (Input)
```typescript
interface ExecRequest {
  task: string;          // Natural language or structured prompt
  adapter?: 'auto' | 'claude' | 'shell'; 
  variables?: Record<string, unknown>; // Context for interpolation
  gate?: {
    enabled: boolean;    // Run FlowMind gate?
    mode: 'strict' | 'relaxed';
  };
}
```

### 3.2 FlowMindGateResult (Output)
```typescript
interface FlowMindGateResult {
  passed: boolean;
  decision: 'allow' | 'deny' | 'propose';
  route?: {
    target: 'lev_exec' | 'claude_code_cli' | 'hybrid';
    factors: ComplexityFactors;
  };
  validation?: ValidationResult; // Output from Validation System
}
```

---

## 4. Implementation Plan

### 4.1 Core Exec Updates
- [ ] **Normalize `ExecRequest`**: Ensure `chat` and `voice` payloads map cleanly to `task`.
- [ ] **Bind Router Policy**: `core/exec` must import `FlowMindRouter` from `core/flowmind` (via `router.policy.ts`).

### 4.2 Validation Gate Loader (FlowMind Workflow)
We will implement the gate loader *as a FlowMind workflow* (`core/validation/workflows/load-gates.flow.yaml`):

```yaml
name: load-validation-gates
description: Load and merge global + project validation gates
trigger:
  manual: true
steps:
  - id: load_global
    action: file.read
    params:
      path: "~/.lev/validation-gates.yaml"
      optional: true
  
  - id: load_project
    action: file.read
    params:
      path: "${PROJECT_ROOT}/.lev/validation-gates.yaml"
      optional: true
      
  - id: merge_config
    action: exec.script
    params:
      lang: typescript
      code: |
        const global = ctx.load_global ?? {};
        const project = ctx.load_project ?? {};
        return mergeDeep(global, project);
  
  - id: configure_system
    action: exec.call
    params:
      target: core.validation.configure
      args: 
        config: "${merge_config.output}"
```

### 4.3 Work Skill Integration
- The `work` skill will invoke this workflow during initialization or `gate:propose-spec` checks.
- It requires `core/exec` to support `flowmind` execution natively (circular dependency resolution: Exec runs FlowMind, FlowMind configures Validation, Validation guards Exec).

### 4.3 Harness Policy Unification
We will replace hardcoded `HarnessType` enums with **FlowMind Policy Documents** (`*.policy.yaml`). This unifies Chat, Voice, Worker, and Dev environments under a single schema.

#### 4.3.1 Proposed Policy Hierarchy
| Policy | Extends | Description | Features / Tricks |
|--------|---------|-------------|-------------------|
| **BaseWorker** | *None* | Abstract root. Defines safe toolset and basic execution loop. | `tools: [read, search]`, `memory: none` |
| **Worker** | BaseWorker | Standard background agent. | `tools: [read, write, search, git]`, `memory: task-scoped` |
| **Chat** | BaseWorker | Interactive session agent. | `memory: session-scoped`, `streaming: true`, `tools: [user-interact]` |
| **Voice** | Chat | Low-latency interactive agent. | `latency: <500ms`, `interruptible: true`, `input: audio-buffer` |
| **Dev** | BaseWorker | CI/CD & Local Dev agent. | `tools: [shell, docker, debug]`, `safety: relaxed` |

#### 4.3.2 Implementation Strategy
- Create `core/flowmind/policies/` to store these definitions.
- Update `ExecRequest` to accept `policy: string` instead of `harness: enum`.
- The `FlowMindRouter` will load the requested policy to configure the execution environment dynamically.

---

## 5. Verification Scenarios

### 5.1 Scenario: Router Policy Enforcement
**Given** a complex task ("Research X, create Y, deploy Z")
**When** sent to `core/exec`
**Then** `router.policy.ts` should detect `complexity > threshold` and route to `lev_exec`.

### 5.2 Scenario: Validation Gate Override
**Given** a project `.lev/validation-gates.yaml` with strict rules
**And** a global `~/.lev/validation-gates.yaml` with loose rules
**When** the loader workflow runs
**Then** the project rules should override/merge with global rules.

### 5.3 Scenario: Chat/Voice Unification
**Given** a voice transcript payload
**When** sent to `exec` as `task`
**Then** it should be processed identically to a typed chat command.

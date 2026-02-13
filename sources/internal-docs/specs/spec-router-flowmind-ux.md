---
title: "Deterministic FlowMind UX Specification"
type: spec
entity: "lev-deterministic-flowmind"
stage: draft
created: "2026-02-10"
author: "codex"
status: draft
epic: lev-aqpc
related: lev-nw2f
---

# Deterministic FlowMind UX Specification

## Executive Summary
This specification defines deterministic execution semantics for FlowMind workflows to ensure reproducible, predictable agent behavior. It establishes UX patterns for expressing determinism constraints in FlowMind YAML, runtime enforcement via the executor, and user-facing feedback when determinism is violated.

## Context

### Problem Statement
FlowMind workflows currently support non-deterministic execution patterns (parallel branches, LLM reasoning, adaptive topology). While powerful for exploratory workflows, these patterns create challenges for:

1. **Reproducibility**: Same input produces different outputs across runs
2. **Auditability**: Execution paths vary, making post-hoc analysis difficult
3. **Testing**: Non-deterministic behavior complicates validation
4. **Compliance**: Regulated domains require predictable behavior trails

### Existing State
- `policy.determinism: true/false` flag exists in step-based format (`StepBasedPolicy`)
- Router's 8-factor complexity model routes workflows but doesn't enforce determinism
- Executor supports parallel steps but doesn't constrain execution order
- No UX feedback when deterministic expectations are violated
- No schema-level validation for determinism-incompatible patterns

### Target State
- Explicit determinism contracts at workflow, step, and trigger levels
- Compile-time and runtime enforcement of determinism constraints
- Clear UX messaging when non-deterministic patterns are detected
- Execution fingerprinting for reproducibility verification
- Deterministic mode for router (`--concurrency=1 --worktree-mode=off`)

## User Scenarios (BDD)

### Scenario 1: Declare Deterministic Workflow
```gherkin
Feature: Deterministic workflow declaration
  As a workflow author
  I want to declare my workflow as deterministic
  So that execution is reproducible across runs

  Scenario: Valid deterministic workflow
    Given a FlowMind workflow with policy.determinism: true
    And all steps are sequential or use deterministic parallel semantics
    When the workflow is compiled
    Then compilation succeeds
    And an execution fingerprint schema is attached

  Scenario: Invalid deterministic workflow
    Given a FlowMind workflow with policy.determinism: true
    And it contains consensus, adaptive, or non-deterministic parallel steps
    When the workflow is compiled
    Then compilation fails
    And error message identifies the violating steps
```

### Scenario 2: Runtime Determinism Enforcement
```gherkin
Feature: Deterministic execution enforcement
  As a workflow operator
  I want runtime to enforce determinism contracts
  So that violations are caught before causing drift

  Scenario: Ordered parallel execution
    Given a deterministic workflow with parallel steps
    When executor runs parallel steps
    Then steps execute in lexicographic ID order
    And output order matches step order
    And execution log includes determinism mode indicator

  Scenario: LLM call with temperature constraint
    Given a deterministic workflow with an agent step
    When the agent step executes
    Then LLM temperature is forced to 0
    And response caching is enabled for identical prompts
    And a warning is logged if model doesn't support temperature=0
```

### Scenario 3: Determinism Violation UX
```gherkin
Feature: Clear UX for determinism violations
  As a workflow author
  I want clear feedback when my workflow violates determinism
  So that I can fix issues before deployment

  Scenario: Compile-time violation
    Given a workflow declares determinism: true
    And contains a step with action: consensus
    When I run flowmind compile
    Then CLI shows: "Determinism violation: step 'gather-opinions' uses consensus (non-deterministic)"
    And exit code is 1
    And fix suggestion is provided

  Scenario: Runtime violation attempt
    Given a deterministic workflow is executing
    And a step attempts to spawn adaptive topology
    When spawn is attempted
    Then spawn is blocked
    And execution fails with "Adaptive topology blocked in deterministic mode"
    And execution log captures the violation
```

### Scenario 4: Execution Fingerprinting
```gherkin
Feature: Reproducibility verification via fingerprinting
  As a compliance officer
  I want to verify identical executions produce identical outputs
  So that I can audit workflow behavior

  Scenario: Fingerprint generation
    Given a deterministic workflow completes successfully
    When execution finishes
    Then a fingerprint is generated containing:
      | Field | Description |
      | workflow_hash | SHA256 of workflow YAML |
      | input_hash | SHA256 of resolved inputs |
      | step_order | Array of step IDs in execution order |
      | output_hash | SHA256 of final outputs |
    And fingerprint is logged to execution record

  Scenario: Fingerprint comparison
    Given two executions of the same deterministic workflow with same inputs
    When fingerprints are compared
    Then workflow_hash, input_hash, step_order MUST match
    And output_hash SHOULD match (warning if not)
```

### Scenario 5: Router Deterministic Mode
```gherkin
Feature: Deterministic routing mode
  As a CLI user
  I want to force deterministic execution via router
  So that I get reproducible results regardless of workflow defaults

  Scenario: Force deterministic via CLI
    Given any FlowMind workflow
    When I run: lev exec --epic=<workflow> --deterministic
    Then router adds concurrency=1 and worktree-mode=off
    And non-deterministic factors are disabled
    And execution uses ordered sequential processing
```

## Behavioral Specification

### Determinism-Incompatible Patterns

The following patterns are **incompatible** with `policy.determinism: true`:

| Pattern | Why Non-Deterministic | Mitigation in Deterministic Mode |
|---------|----------------------|----------------------------------|
| `consensus` | Multiple LLM opinions synthesized | Block at compile time |
| `adaptive` | Topology changes mid-execution | Block at compile time |
| `streaming` | Confidence-based termination | Fixed iteration count |
| `parallel` (unordered) | Race conditions | Ordered by step ID |
| `memory.federated` | Cross-agent state sharing | Single-agent namespace |
| `spawn` (dynamic) | Unknown agent count | Static spawn only |
| LLM `temperature > 0` | Probabilistic sampling | Force temperature=0 |

### Determinism-Compatible Patterns

These patterns are safe for deterministic workflows:

| Pattern | Deterministic Semantics |
|---------|------------------------|
| `sequential` | Execute in declaration order |
| `parallel` (ordered) | Execute in step ID order, collect in order |
| `feedback` (bounded) | Fixed max_iterations, no early exit |
| `memory.local` | Session-scoped, single agent |
| `spawn` (static) | Pre-declared agents only |
| LLM `temperature=0` | Deterministic sampling |

### Policy Schema Extension

```yaml
# Extended policy schema for determinism
policy:
  determinism: boolean | DeterminismConfig

DeterminismConfig:
  enabled: boolean
  enforce_at: compile | runtime | both  # default: both
  parallel_order: lexicographic | declaration  # default: lexicographic
  llm_temperature: 0  # forced
  cache_identical_prompts: boolean  # default: true
  fingerprint: boolean  # default: true when enabled=true
  violations:
    on_detect: fail | warn | log  # default: fail
```

### Execution Fingerprint Schema

```yaml
# Execution fingerprint for reproducibility verification
fingerprint:
  version: "1.0"
  workflow:
    name: string
    hash: string  # SHA256 of normalized YAML
  execution:
    id: string  # UUID
    timestamp: ISO8601
    deterministic: true
  inputs:
    hash: string  # SHA256 of resolved inputs
    keys: string[]  # Input variable names
  steps:
    order: string[]  # Step IDs in execution order
    hashes: Record<string, string>  # Step ID → output hash
  outputs:
    hash: string  # SHA256 of final outputs
    keys: string[]  # Output variable names
  environment:
    node_version: string
    platform: string
    lev_version: string
```

### Compiler Validation Rules

The FlowMind compiler MUST validate determinism constraints:

1. **Parse Phase**: Detect `policy.determinism: true`
2. **Validate Phase**: Scan steps for incompatible patterns
3. **Error Phase**: Report violations with step IDs and fix suggestions
4. **Output Phase**: Attach fingerprint schema if valid

Validation errors format:
```
DETERMINISM_VIOLATION: step '<step_id>' uses '<pattern>'
  Location: line <n>, column <m>
  Pattern: <pattern_name> is non-deterministic because: <reason>
  Fix: <suggestion>
```

### Executor Enforcement

The FlowMind executor MUST enforce determinism at runtime:

1. **Pre-execution**: Validate policy.determinism flag
2. **Parallel steps**: Execute in `step.id` lexicographic order
3. **LLM calls**: Force temperature=0, enable response caching
4. **Spawn attempts**: Block dynamic spawns, allow static only
5. **Post-execution**: Generate and log fingerprint

### UX Messaging

#### CLI Output

```bash
# Compile-time success
$ flowmind compile workflow.yaml
✓ Workflow 'my-workflow' compiled (deterministic mode)
  Fingerprint schema attached
  Steps validated: 5/5

# Compile-time violation
$ flowmind compile workflow.yaml
✗ Determinism violation in 'my-workflow'

  Step 'gather-opinions' uses consensus pattern
    → consensus is non-deterministic (multiple LLM outputs synthesized)
    → Fix: Remove consensus or set policy.determinism: false

  Step 'adapt-strategy' uses adaptive pattern
    → adaptive is non-deterministic (topology changes at runtime)
    → Fix: Use static workflow topology

# Runtime execution
$ lev exec --epic=my-workflow --deterministic
▸ Executing in deterministic mode
  ├─ Step order: init → process → validate → output
  ├─ LLM temperature: 0 (forced)
  └─ Fingerprint: 7a3f2e...

✓ Execution complete
  Fingerprint: sha256:7a3f2e8c...
  Reproducibility: verified
```

#### Statusline Integration

```
[flowmind:deterministic] my-workflow ▸ step-3/5 ◆ fingerprint:7a3f
```

## Contract

### Dependencies
- `@lev-os/flowmind` parser, compiler, executor
- `@lev-os/config` for determinism defaults
- `@lev-os/events` for fingerprint logging
- CLI harness for UX messaging

### Integration Points
- `core/flowmind/src/parser.ts`: Extend policy parsing
- `core/flowmind/src/compiler.ts`: Add determinism validation
- `core/flowmind/src/executor.ts`: Add runtime enforcement
- `core/flowmind/src/router.ts`: Add `--deterministic` flag handling
- `core/flowmind/src/schema.ts`: Add DeterminismConfig type

### Breaking Changes
- Workflows declaring `policy.determinism: true` with incompatible patterns will fail compilation
- Parallel steps in deterministic workflows will execute in different order (ID-sorted vs race)

## Implementation Guidance

### Recommended Skills
- `work`
- `lev-builder`
- `bd`

### Team Structure
Medium-complexity serial execution (one primary workstream with validation gates).

### Workstreams

1. **Schema & Validation** (Priority 1)
   - Extend `StepBasedPolicy` with `DeterminismConfig`
   - Add validation rules to compiler
   - Implement compile-time error messaging

2. **Executor Enforcement** (Priority 2)
   - Add determinism mode to executor context
   - Implement ordered parallel execution
   - Add LLM temperature forcing
   - Block incompatible patterns at runtime

3. **Fingerprinting** (Priority 3)
   - Define fingerprint schema
   - Implement hash generation
   - Add fingerprint logging to execution result
   - Add fingerprint comparison utility

4. **UX & CLI** (Priority 4)
   - Add `--deterministic` flag to router
   - Implement statusline integration
   - Add fingerprint display in CLI output

## Test Coverage

### Unit Tests
- Policy parsing with DeterminismConfig
- Validation rule detection for each incompatible pattern
- Parallel step ordering (lexicographic sort)
- Fingerprint hash generation

### Integration Tests
- Compile workflow with determinism violations → error
- Execute deterministic workflow → ordered execution
- Execute with `--deterministic` flag → forced mode

### E2E Verification

1. **Triad A: Compile-Time Validation**
   - Unit: `pnpm --filter @lev-os/flowmind test -- determinism`
   - Integration: Compile fixture with violations → expect errors
   - E2E: `flowmind compile examples/deterministic.flow.yaml` passes

2. **Triad B: Runtime Enforcement**
   - Unit: Executor determinism enforcement tests
   - Integration: Execute parallel steps → verify order
   - E2E: `lev exec --epic=fixture --deterministic` twice → same fingerprint

3. **Triad C: Reproducibility**
   - Unit: Fingerprint generation tests
   - Integration: Same inputs → same fingerprint
   - E2E: Compare fingerprints across runs

## Success Criteria
- 100% of incompatible patterns detected at compile time
- Deterministic parallel execution order is reproducible
- Fingerprints match for identical inputs and workflows
- CLI provides actionable error messages for violations
- `--deterministic` flag works with existing workflows

## Open Questions
- Should deterministic mode be the default for CI/CD pipelines?
- How should external tool calls (shell commands) be handled for reproducibility?
- Should fingerprints be stored in BD as audit records?
- What is the performance overhead of fingerprint generation?

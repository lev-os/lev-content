---
module: "config"
design_refs:
  - "docs/design/contracts.md"
  - "docs/design/core-other.md"
code_refs:
  - "src/core/config/**"
validation_gates:
  - "gate:config-schema-validity"
---

entity: "lev-config-sprawl-governance"
stage: crystallized
created: "2026-02-09"
author: "codex"
status: draft
---

# Lev Config Sprawl Elimination and Fractal Governance — Behavioral Specification

## Executive Summary
This specification defines a strict, contract-first configuration model for Lev to eliminate config sprawl and make install/runtime behavior deterministic. It separates host-level `system` controls from user-level `global` defaults, enforces scope-safe writes, and standardizes module config as executable contracts rather than narrative notes.

## Context
Current architecture already documents a fractal chain and XDG posture, but implementation and module surfaces allow drift and ambiguity. This spec hardens precedence, write permissions, and schema validation so `lev install`, module install surfaces, and runtime config resolution stay aligned.

### Existing State
- Architecture describes fractal config layering and local override semantics in docs.

- Config loader currently merges multiple surfaces (including routing/profile), but permissive parsing and mixed shapes can hide drift.

- Installer currently sets broad defaults and sync behavior in one command path, blending policy and runtime concerns.

- Module `config.yaml` files vary in strictness; many contain mixed metadata + runtime shape without enforced schemas.

### Target State
- Single authoritative config overlay contract with explicit scope taxonomy:
  1. `system` (host services: launchd/systemd, daemon boot wiring)
  2. `global` (user policy defaults: for example TDD preference)
  3. `project` (repo-local behavior)
  4. `module` (module declarations and capabilities)
  5. `env` (runtime ephemeral overrides)

- Installer and module hooks are scope-safe and idempotent by default.

- Every module config is schema-validatable and machine-enforced.

- Config write operations emit `LevEvent` audit records.

## User Scenarios (BDD)

### Scenario 1: Deterministic Scope Precedence
```gherkin
Feature: Fractal config precedence with explicit scope boundaries
  As a Lev framework developer
  I want config resolution to be deterministic across scopes
  So that behavior is predictable and reproducible across machines

  Scenario: Resolve with all scopes present
    Given system, global, project, module, and LEV_* env values exist for the same key
    When Lev resolves runtime configuration
    Then precedence MUST be system < global < project < module < env
    And the resolved output MUST include source provenance metadata
```

### Scenario 2: Safe Installer Writes
```gherkin
Feature: Scope-safe install writes
  As an installer module author
  I want install writes constrained by declared scope
  So that no hidden host-level mutations occur

  Scenario: Module requests system write without elevation
    Given a module install surface declares a write target in system scope
    When lev install runs without explicit elevation and user confirmation
    Then the write MUST be denied
    And installer output MUST report the denied write and reason
```

### Scenario 3: Contract-First Module Config
```gherkin
Feature: Strict module config schema
  As a core module maintainer
  I want module config files to be validated strictly
  So that free-form note-style YAML does not enter runtime paths

  Scenario: Unknown key in strict namespace
    Given a module config contains unknown keys in a strict namespace
    When config validation executes
    Then validation MUST fail with actionable errors
    And runtime loading MUST not silently coerce unknown fields
```

### Scenario 4: Idempotent Install Re-run
```gherkin
Feature: Idempotent install behavior
  As a Lev user
  I want repeated installs to be safe
  So that reruns do not produce drift

  Scenario: Re-run install with unchanged inputs
    Given lev install has completed successfully once
    When lev install is executed again with identical inputs
    Then no additional config mutations occur
    And generated outputs remain byte-equivalent
```

### Scenario 5: Config Audit via LevEvent
```gherkin
Feature: Auditable config mutation lifecycle
  As a platform operator
  I want all config mutations evented
  So that post-change investigations are possible

  Scenario: Persisted config write
    Given a config write is accepted
    When the write is committed
    Then Lev MUST emit a LevEvent with type/source/ts/id/data
    And data MUST include scope, key path, and old/new hash
```

## Behavioral Specification

### Inputs
- XDG/root config files and scoped overlays.

- Project config files and optional local overrides.

- Module `config.yaml` declarations plus optional `install.module.ts` export object.

- CLI flags (`--wizard-mode`, `--human`, `--no-wizard`, explicit overrides).

- `LEV_*` environment overrides.

### Processing
1. Parse and validate each scope against strict schema.

2. Merge by canonical precedence order (system → global → project → module → env).

3. Evaluate install write intents against declared `install.writes` and scope guardrails.

4. Block unauthorized writes; apply authorized writes transactionally.

5. Emit `LevEvent` records for accepted writes and write decisions.

6. Return resolved config + provenance + mutation summary.

### Outputs
- `ResolvedConfig` object with explicit source metadata.

- Installer result summary including accepted/denied writes.

- Persistent scoped config updates (only where authorized).

- Event stream records for config mutation lifecycle.

### Performance
- Config resolution: p95 under 200ms for standard project size.

- Installer scope-check phase: p95 under 100ms excluding I/O-heavy tasks.

- No network dependency in core resolution path.

## Contract

### Dependencies
- `@lev-os/config` loader and type layer.

- `@lev-os/build` install command and wizard runner.

- Module `config.yaml` declarations across `core/*` and target plugins.

- Event bus contract using `LevEvent`.

### Integration Points
- `core/config/src/config-loader.ts` as authoritative resolver implementation.

- `core/build/src/commands/install.ts` for write authorization and wizard behavior.

- `docs/01-architecture.md` and `AGENTS.md` for canonical contract documentation.

- Module install surfaces: `config.yaml` baseline with optional `install.module.ts` object export.

### Breaking Changes
- Implicit host-level writes become explicit elevated operations only.

- Unknown keys in strict namespaces are rejected.

- Legacy note-style YAML accepted only in migration mode, not strict mode.

## Implementation Guidance

### Recommended Skills
- `work`

- `research`

- `lev-builder`

- `lev-install`

- `bd`

### Team Structure
- Medium-complexity parallel execution with three workstreams.

### Workstreams
1. Resolver Contract Hardening
- Owner: config maintainer.
- Scope: precedence contract, strict parsing, provenance emission.

2. Installer Scope Enforcement
- Owner: build/install maintainer.
- Scope: write authorization matrix, elevation confirmation, idempotent rerun guarantees.

3. Module Contract Migration
- Owner: platform/core integration maintainer.
- Scope: module schema normalization, strict namespaces, migration adapters for legacy YAML.

## Test Coverage

### Unit Tests
- Scope precedence matrix (all five scopes).

- Write guard policy checks (allowed/denied by scope + elevation).

- Strict schema behavior for unknown keys and invalid types.

### Integration Tests
- `lev install` with module-declared install surfaces (`config.yaml` + optional hook object).

- Resolver + installer end-to-end write authorization path.

- LevEvent emission contract for accepted/denied write decisions.

### E2E Verification
- Deterministic test triads (run in order):

1. Triad A: Scope Resolution Determinism
- Unit: `pnpm --filter @lev-os/config test`
- Integration: `pnpm test:vitest -- core/config`
- E2E: run `lev install --wizard-mode=none` twice in fixture repo and assert zero diff.

2. Triad B: Install Write Safety
- Unit: `pnpm --filter @lev-os/build test`
- Integration: install fixture with forbidden `system` write without elevation must fail.
- E2E: install fixture with explicit elevated confirmation must pass and emit events.

3. Triad C: Module Contract Strictness
- Unit: schema validation tests for module namespaces.
- Integration: strict-mode load rejects unknown keys in module config.
- E2E: full workspace validate (`pnpm test:all`) with strict config mode enabled.

## Success Criteria
- Zero implicit writes to `system` scope.

- 100% of touched modules have schema-valid install/config surfaces.

- Resolver precedence behavior is deterministic and provenance-visible.

- Install rerun with identical inputs yields byte-equivalent config artifacts.

- All persisted config writes emit valid `LevEvent` audit entries.

## Open Questions
- Should `routing.yaml` remain a distinct surface or fold into the strict global scope model?

- What is the migration sunset window for legacy permissive YAML parsing?

- Should strict mode be default-on immediately or phased by module readiness?

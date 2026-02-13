# Lev Development Process

**Version**: 2.0.0
**Status**: CANONICAL

---

## How This Relates to the Work Skill

The universal work lifecycle is defined in `~/.agents/skills/work/SKILL.md`. That FSM (DISCOVER > ALIGN > RESEARCH > PROPOSE > SPEC > EXECUTE > VALIDATE > EMIT > LEARN) governs all work across all projects. Beads (`br`) is the artifact backend. Guard scoring runs constantly.

**This document does NOT redefine that FSM.** It defines Lev-specific conventions that plug into it:

1. Spec standards by component tier (how much rigor each type of work needs)
2. Layout alignment rules (Lev-specific directory checks)
3. Shovel-ready criteria per tier
4. Where artifacts go in this monorepo

---

## 1. Spec Standards by Tier

Not everything needs 242KB of specs. A Rust filesystem crate touching data integrity is not a TypeScript CLI command. The tier determines how much documentation, testing, and validation is required before work is "shovel ready."

### Tier Classification

| Tier | What | Examples | Spec Rigor |
|---|---|---|---|
| **T1: Kernel** | Rust crates touching data integrity, concurrency, filesystem, security | `lev-agentfs`, `lev-reactive`, sandbox, MVCC | Formal |
| **T2: Infrastructure** | Rust crates for UI, platform, rendering, cross-cutting libraries | `lev-tui-*`, `lev-desktop`, `lev-ui-universal` | Standard |
| **T3: Integration** | TypeScript/Python modules, CLI commands, MCP servers, plugin adapters | `core/cli`, `core/flowmind`, `plugins/platforms/*`, MCP tools | Brief |
| **T4: Community/Tooling** | Standalone OSS packages, skills, dev utilities, scripts | `community/*`, `tooling/*`, workflow skills | Minimal |

### Per-Tier Requirements

#### T1: Kernel (Formal)

Shovel-ready means ALL of:

- [ ] **Formal spec** (`docs/SPEC.md`) — behavioral contract, schema definitions, operation semantics. 5KB minimum. Updated in same PR as code changes.
- [ ] **Feature parity tracking** (`docs/FEATURE_PARITY.md`) — quantitative: X/Y features per domain, overall percentage. Updated every release.
- [ ] **Integration spec** (if Lev-specific extensions exist) — what Lev adds on top of upstream, API surface, platform matrix.
- [ ] **Conformance tests** — golden fixture comparison or property-based tests. Schema/behavior changes must break goldens explicitly.
- [ ] **Adversarial validation gates** — for each phase of work, define what must pass before proceeding. Corruption injection, conflict simulation, edge case verification.
- [ ] **Changelog** (`docs/CHANGELOG.md`) — versioned, every release.
- [ ] **Testing guide** (`docs/TESTING.md`) — how to run conformance, benchmarks, and integration tests.
- [ ] **CI gate**: `cargo fmt --check && cargo check --all-targets && cargo clippy -- -D warnings && cargo test --workspace`

FrankenFS reference point: they have 437KB of specs for a 37%-complete project. That's excessive. But the categories are right: spec, parity, conformance, testing guide, changelog. Aim for 20-50KB total for a T1 crate.

#### T2: Infrastructure (Standard)

Shovel-ready means ALL of:

- [ ] **API spec** — public trait/struct documentation, either in `docs/` or as rustdoc. Not a full behavioral spec, but the contract surface must be documented.
- [ ] **Changelog** — versioned.
- [ ] **Testing guide** — how to run tests, what they cover.
- [ ] **README** — purpose, usage, architecture overview.
- [ ] **CI gate**: same as T1.

#### T3: Integration (Brief)

Shovel-ready means ALL of:

- [ ] **README** — purpose, usage, configuration.
- [ ] **Integration tests** — at least one happy path, one error path.
- [ ] **Changelog** — can be inline in README for small modules.
- [ ] **CI gate**: typecheck + test pass.

#### T4: Community/Tooling (Minimal)

Shovel-ready means ALL of:

- [ ] **README** — install instructions, usage.
- [ ] **LICENSE** (MIT default).
- [ ] **At least one test or usage example**.
- [ ] Marketplace metadata if publishing (`.claude-plugin/marketplace.json`).

### Quick Decision Table

| Question | T1 | T2 | T3 | T4 |
|---|:---:|:---:|:---:|:---:|
| Does it touch data integrity? | Yes | No | No | No |
| Is it Rust with unsafe-adjacent concerns? | Yes | Maybe | No | No |
| Can a bug here corrupt user data? | Yes | No | No | No |
| Does it have a FUSE/kernel/syscall surface? | Yes | No | No | No |
| Is it a library other crates depend on? | Yes | Yes | Maybe | No |
| Does it ship as a standalone binary? | Maybe | Maybe | Maybe | No |
| Is it a CLI command or plugin? | No | No | Yes | Maybe |
| Is it an OSS community package? | No | No | No | Yes |

---

## 2. Layout Alignment (Lev-Specific)

Before starting work that creates or moves modules, verify against the canonical structure.

### Checks

1. **New Rust crate?** — Goes under `crates/`. Follows naming in `AGENTS.md` 3.7.
2. **New TS/core module?** — Goes under `core/`. Check `docs/01-architecture.md` for canonical layout.
3. **New plugin?** — Goes under `plugins/`. Needs `leviathan` key in `package.json`.
4. **New community package?** — Goes under `community/`. Needs own git repo as submodule.
5. **Renaming/moving?** — grep all docs, AGENTS.md, .lev/pm/ for old name. Update all references.

See `AGENTS.md` sections 3-3.7 for the full directory taxonomy and ownership map.

---

## 3. Artifact Locations

All artifacts are beads (`br create`). Markdown views are rendered by hooks for human readability. File locations for rendered views:

| Artifact Type | Bead Type | Rendered View Location |
|---|---|---|
| Report | `custom:report` | `.lev/pm/reports/` |
| Proposal | `custom:proposal` | `.lev/pm/proposals/` |
| Spec | `custom:spec` | `.lev/pm/specs/` |
| Handoff | handoff bead | `.lev/pm/handoffs/` |
| Design | N/A (legacy) | `.lev/pm/designs/` |
| Plan | N/A (legacy) | `.lev/pm/plans/` |
| Decision | N/A (legacy) | `.lev/pm/decisions/` |
| Analysis | N/A (legacy) | `.lev/pm/analysis/` |

Crate-specific documentation (specs, changelogs, parity tracking) lives in `<crate>/docs/`, NOT in `.lev/pm/` or monorepo `docs/`. See `docs/README.md` for the full documentation layer breakdown.

---

## 4. Validation Gate Quick Reference

The full gate definitions live in `~/.agents/skills/work/references/gates.md`. Summary for Lev development:

| Transition | What Must Pass |
|---|---|
| Idea to spec | Guard score <=30%, alignment check, prior art search |
| Spec to execute | 16-point catastrophic gate (see `work/SKILL.md` VALIDATE section). Never skip. |
| Execute to done | Tests pass, docs updated, BD task closed, feature parity updated (if T1/T2) |
| Session end | Handoff emitted, learnings captured (`cm reflect`), `br sync`, `git push` |

---

## 5. Execution Tiers (from Work Skill)

The `work` skill classifies by shearing layer. Here's how that maps to Lev components:

| Classification | Tier | Lev Examples | Pattern |
|---|---|---|---|
| Stuff + L3 + CDO | Direct | Env var change, copy tweak, config update | Just do it |
| Space Plan + L2 | Brief | New CLI command, plugin adapter, skill | 5-bullet spec then execute |
| Skin/Services + L1-L2 | Standard | TUI redesign, MCP server, new plugin system | Proposal then execute then review |
| Structure + L1 | Deliberate | New core module, event schema change, SDK redesign | Proposal then approval then execute then review |
| Site + L0-L1 | Formal | AgentFS schema change, MVCC, new filesystem, auth model | Proposal then approval then phased execution then multi-gate review |

---

## Revision History

### Version 2.0.0 (2026-02-11)

- Replaced 7-phase epic refinement with spec standards by tier
- Deferred FSM lifecycle to `work/SKILL.md` (single source of truth)
- Added tier classification system (T1-T4) with shovel-ready checklists
- Removed Ralph/daemon automation (deprecated)
- Added execution tier mapping to Lev components
- Added artifact location table with beads integration

### Version 1.0.0 (2026-01-27)

- Initial epic refinement process (7 phases)

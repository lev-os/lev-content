---
title: "From Prototype to Production in One Ratchet: The Gate Lifecycle"
description: "The gate ratchet: aspirational to declared to enforced to auto-enforced. Why it only goes one way. How it prevents regression and makes quality irreversible."
tags: [gates, ratchet, validation, quality, constraint-engineering, lifecycle]
date: 2026-04-10
series: leviathan-engineering
---

# From Prototype to Production in One Ratchet: The Gate Lifecycle

Every software project has standards. Most of them are lies.

Not intentional lies. Aspirational lies. "We do code review." Sometimes. "We maintain test coverage." When we remember. "We follow the style guide." Except when we are in a hurry, which is always.

The problem is not that teams lack standards. The problem is that standards degrade. Someone merges without review because it is Friday. Test coverage drops from 80% to 72% and nobody notices. The style guide accumulates exceptions until it describes an alternate reality.

Leviathan's gate ratchet is a mechanism designed to make this degradation structurally impossible. It is a one-way lifecycle for validation rules, and it is one of the most consequential design decisions in the system.

## The Four States

Every validation gate in Leviathan exists in exactly one of four states:

**Aspirational.** The target state. Not yet checked. This is where standards start: someone writes down what they want to be true. "No file should exceed 500 lines." "Every module should have a config.yaml." "All active code should reference only current module names." At this stage, the gate is a statement of intent. It exerts gravitational pull on development decisions but does not enforce anything.

**Declared.** Manual review only. No automated checker yet, but someone is responsible for verifying the rule during code review. The gate has moved from "we should do this" to "we check for this." The difference is accountability. A declared gate appears in review checklists. It has an owner.

**Enforced.** An automated check exists and runs. Violations either block the operation or produce warnings. This is the critical transition. When a gate moves from declared to enforced, a human judgment call becomes a machine-verifiable property. The check runs every time, not just when someone remembers.

**Auto-enforced.** The check runs and an auto-fix flow resolves violations without human intervention. The gate does not just catch problems -- it repairs them. A formatting gate that auto-enforces runs the formatter and fixes violations before they ever reach a human reviewer.

## The One-Way Property

Here is the rule that makes the ratchet work: **gates only move forward.**

Aspirational becomes declared. Declared becomes enforced. Enforced becomes auto-enforced. Never the reverse.

You cannot take an enforced gate and demote it to declared because enforcement is inconvenient. You cannot take a declared gate and push it back to aspirational because the team decided they do not care anymore. The ratchet only clicks in one direction.

This derives from C4 Ratchet Convergence: "State transitions converge toward attractor basins. Rollback requires explicit compensation. Append-only." In plain language: you cannot silently undo progress. If you truly need to remove a gate, you must create an explicit compensation -- a documented decision with a reason, an owner, and an audit trail. You cannot just quietly delete a line from a YAML file.

## Why One-Way Matters

Consider what happens without the one-way property.

A team enforces a "no circular dependencies" gate. It catches three violations in the first week. The violations are painful to fix because they require refactoring. Someone proposes "relaxing" the gate to declared, meaning it becomes a suggestion instead of a requirement. The team agrees because the refactoring is blocking a deadline.

Now the gate is declared. Review catches circular dependencies sometimes. After a few weeks, reviewers stop checking because nobody remembers it was ever enforced. The gate drifts to aspirational. New circular dependencies accumulate. Six months later, the codebase has 15 circular dependency chains and the cost to fix them is prohibitive.

This is not a hypothetical. This is the lifecycle of every standard in every codebase that lacks a ratchet mechanism. Standards erode. They erode because every local decision to relax a standard is rational, and the cumulative effect is catastrophic.

The one-way ratchet prevents this. The "no circular dependencies" gate stays enforced. The team has to fix the violations. It is painful in week one. By month six, circular dependencies do not exist, and nobody even thinks about them. The quality is irreversible.

## What This Looks Like in Practice

Leviathan's gate definitions live in `dna/gates.yaml` and `.lev/validation-gates.yaml`. Here is a real example:

```yaml
no_commands_dir:
  status: enforced
  category: architecture
  rule: "Use src/handlers/ not src/commands/. 'command' implies CLI, breaks surface-agnostic design."
  scope: "core/**/src/, plugins/**/src/"
  check: |
    ! find core/ plugins/ -type d -name commands -path '*/src/*' | grep -q .
  on_fail: block
  violations_known: 0
```

This gate is enforced. It has an automated check. Violations block the operation. The rule is clear, the rationale is documented, and the scope is explicit.

Here is a gate at an earlier stage:

```yaml
no_export_star:
  status: aspirational
  category: architecture
  rule: "No export * re-exporting another package's types. Each package owns its API surface."
  scope: "core/*/src/index.ts"
  violations_known: 0
```

This gate is aspirational. It represents a design direction but does not have an automated checker yet. When someone writes the checker, it will move to declared (manual review via the checker output) and then to enforced (automated, blocking).

The system currently has 61 module gates spanning architecture, quality, runtime, and DNA compliance categories -- all cross-referenced with 61 specs in `docs/specs/README.md`. Each gate tracks its status, and the status only moves forward. 96% compliance across enforced gates. The remaining 4% are aspirational gates actively climbing the ratchet.

## The Violation Ledger

Gates also track `violations_known` -- the count of existing violations at the time the gate was created or promoted. This is critical for practical adoption.

When you enforce a gate on an existing codebase, you often have pre-existing violations. The violation ledger acknowledges them explicitly. "This gate has 11 known violations as of enforcement date." The gate prevents new violations from being introduced while the team works through the backlog.

This prevents the "all or nothing" problem that kills most quality initiatives. You do not have to fix every violation before you can enforce the gate. You enforce the gate with a known violation count, then ratchet that count down over time. The count, like the gate status, only moves in one direction.

## Gate Categories

Leviathan organizes gates into five evaluation types, each answering a different question:

- **Scope gates**: Is this within the project or workstream boundary?
- **Safety gates**: Does this involve secrets, destructive actions, or sensitive data?
- **Feasibility gates**: Are the required tools available? Are permissions held?
- **Standards gates**: Does this satisfy rules, formatting, and invariants?
- **Progress gates**: Did the last tick measurably advance the work?

The progress gate deserves special attention. It answers the question: "Did something actually happen?" This prevents busy-work cycles where an agent (or a human) spins without producing artifacts. Every tick must produce a concrete artifact, advance a bead, or move a workstream forward. If it does not, the tick fails the progress gate.

## From Manual to Automatic: The Enforcement Progression

The most practical benefit of the ratchet is that it provides a clear path from "we care about this" to "this happens automatically."

Stage 1: Write the rule as an aspirational gate. This costs almost nothing. You are recording a design intent.

Stage 2: Assign an owner and move to declared. Someone reviews for compliance manually. This costs review time but catches violations.

Stage 3: Write an automated check and move to enforced. This costs development time once but eliminates ongoing review burden.

Stage 4: Add an auto-fix flow and move to auto-enforced. The gate resolves its own violations. This is the terminal state -- the quality concern is permanently resolved.

The ratchet makes each stage a one-way commitment. You do not backslide. You invest once and keep the gains permanently.

## The Competitive Moat

Here is the long game. Every gate that reaches enforced or auto-enforced becomes part of the system's constraint surface. That surface only grows. Every exploit hardens the system. Every violation caught becomes a permanent check.

Over time, a system with a gate ratchet accumulates a constraint surface that would take a competitor years to replicate. Not because the individual rules are secret -- they are in a YAML file. But because the discipline of one-way enforcement compounds. Twelve months of one-way ratcheting produces a quality baseline that is practically impossible to achieve through periodic cleanup sprints.

The ratchet is not a quality mechanism. It is strategy. It makes quality irreversible, and irreversible quality is a moat. Constraints are the product. The ratchet is the business model.

---

*This article is part of the Leviathan engineering series exploring the design principles behind the universal agent runtime. Leviathan is open source at [github.com/lev-os/lev](https://github.com/lev-os/lev).*

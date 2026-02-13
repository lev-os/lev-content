# Leviathan North Star

> **One runtime. Every agent. Every surface.**

---

## The Idea

Software agents are trapped in chat windows. They type text, you type text back. The interface hasn't changed since IRC.

Leviathan breaks this loop. Agents don't chat — they *act* through any surface: a browser button, a voice command, a VR gesture, a Stream Deck key, a mobile notification. **AgentPing** makes every surface an interaction point. **AgentLease** makes every interaction accountable. **FlowMind** makes every decision auditable.

This isn't an agent framework. It's the operating system for agent-human symbiosis.

---

## Three Systems, One Runtime

| System | What It Is | Why It Exists |
|--------|-----------|---------------|
| **LEV** | Orchestration runtime — lifecycle graph, policy evaluation, worker execution | Agents need a substrate that enforces constraints, routes intent, and remembers everything |
| **AgentPing** | Human-loop interaction protocol — component kit, dashboard surfaces, approval UX | Humans need to interact with agents through *surfaces*, not chat. Any surface can be an AgentPing surface |
| **AgentLease** | Scoped permission grants — step-up auth, lease issuance, expiration-based accountability | Agents must earn trust. Leases expire. Permissions are scoped. Nothing runs forever unchecked |

```
Human Intent
    │
    ▼
AgentPing (any surface)
    │
    ▼
FlowMind (parse → route → constrain → enforce)
    │
    ▼
LEV Exec (SDK → CLI → MCP → poly)
    │
    ▼
AgentLease (scoped grant → monitored execution → expiration)
```

---

## FlowMind Is the Substrate

Everything flows through FlowMind. Not around it. Not beside it. Through it.

- **Intent** flows in as natural language → parsed into typed actions
- **Constraints** are loaded as immutable kernel declarations → fail-closed enforcement
- **Memory** decays naturally → strong memories persist, rejected ones fade
- **Events** are emitted at every boundary → durable, replayable, auditable

FlowMind isn't a workflow engine. It's the programmable substrate the entire system is built on. Agent behaviors, entity lifecycles, policy evaluation, memory decay — all expressed as FlowMind declarations.

---

## What Already Exists

**FlowMind compiler** — YAML → 6 targets (smartdown, system prompt, prose, hooks, schedule, prompt gen). 128 tests passing. E2E operational.

**`@lev-os/exec` SDK** — Universal execution API with FlowMind gate at the front. Fail-closed by default. 20 tests passing. CLI/MCP/poly surfaces generated from the SDK.

**60+ CLI commands** across 8 modules. 9 execution adapters. 10 search backends. 7 entity schemas with lifecycle FSMs.

**AgentPing** — Hexagonal core with 6 active UI surfaces. Sofia-first rendering. GenUI research promoted into implementation specs. Dashboard runner with auto-restart, port management, health monitoring.

**Memory decay** — Exponential decay, rejection acceleration, usage refresh, automated pruning. 17 tests.

---

## What We're Building Toward

**Agents that earn autonomy.** An agent starts with leased permissions. As it succeeds, leases extend. As it fails, leases expire. No hardcoded trust levels — trust is earned through demonstrated competence.

**Surfaces that match the moment.** A critical deploy approval shows up as a blocking modal. A routine code review arrives as a notification badge. A brainstorm unfolds in a canvas. The agent chooses the surface intensity based on the decision weight.

**Memory that ages like wine.** Good patterns strengthen. Rejected approaches decay. Stale knowledge fades. The system remembers what *works*, not just what *happened*.

**Constraints that never bend.** Fail-closed, not fail-open. If the kernel can't evaluate a policy, the action is denied. No inference, no repair, no silent failures. The ratchet only tightens.

---

## Principles

1. **FlowMind as substrate** — everything flows through it, not around it
2. **SDK-first** — TypeScript SDK → CLI, MCP, and cross-platform SDKs generated from it
3. **Fail-closed** — cannot evaluate → DENY
4. **Surfaces, not chat** — agents interact through action surfaces, not text streams
5. **Leases, not permissions** — everything expires, trust is earned
6. **Contract-first** — types and interfaces before implementation
7. **Hexagonal** — ports, adapters, business logic in core

---

*Last validated: 2026-02-12 · Branch: `feat/memory-as-flowmind`*

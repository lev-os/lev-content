# The Tick: Autonomous Loops & Scheduling

> **Source:** `00-vision.md` lines 98-117, 114-117, 336 (voice transcript)
> **Status:** ephemeral → needs spec (but NOT highest priority per user)
> **Layer:** Structure (scheduling primitive)
> **Interview notes:** User says the Tick is not the highest priority spec — graph primitives need to be worked out first. The Tick already appears in `docs/design/graph-primitives.md` as INGEST→OBSERVE→PROPOSE→GATE→APPLY→UPDATE→EMIT but has no implementation spec.

## Core Concept

The Tick is the heartbeat of the autonomous system. Currently, the user IS the heartbeat ("I'm the heartbeat right now. I have to do it."). The goal is to make progress happen without the user.

### The Loop
```
INGEST → OBSERVE → PROPOSE → GATE → APPLY → UPDATE → EMIT
  ↑                                                    │
  └────────────────────────────────────────────────────┘
```

- Everything is a loop — key insight
- A cron job creates the tick
- Each tick traverses pending proposals, evaluates gates, applies approved patches

### Entity Classification as Input
When the system ingests a message, it performs entity classification:
- Chat message (primitive)
- Session
- Agent
- Time elapsed / knowledge of time
- Ideas → knowledge graph nodes
- Capabilities → what the system can do

### Intent vs Goal
> "I don't want to be goal-driven, I want to be intent-driven. When I train the model, it's not reward-based, it's intent-based."

### Stream of Consciousness Router
- Human interaction is stream of consciousness — all over the place
- Different time horizons, different layers of context
- All nodes on a graph with a "nucleus" (essence of a thing)
- The tip of the spear for any agentic framework is the action surface
- Multiple workstreams to manage simultaneously
- Entities moving through a finite state machine

### Entities Have Lifecycle Constraints
> "There has to be a concrete way that it can go through reality. It doesn't just exist as a markdown file. You have to create it, update it, delete it. It goes stale. That's why we need the heartbeat."

## Relationship to Existing Code
- `docs/design/graph-primitives.md` — Tick primitive defined but unimplemented
- `work` skill FSM — the manual version of this (DISCOVER→ALIGN→RESEARCH→PROPOSE→SPEC→EXECUTE→VALIDATE→EMIT→LEARN)
- The work skill loop IS the tick, manually invoked

## Priority
Per interview: **NOT highest priority.** Graph primitives come first. The Tick is a natural consequence of having graph primitives + FlowMind YAML in place.

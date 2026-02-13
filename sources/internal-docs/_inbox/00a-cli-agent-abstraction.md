# CLI Agent Abstraction & Terminal Interaction Model

> **Source:** `00-vision.md` lines 1-25 (voice transcript)
> **Status:** ephemeral → needs DISCOVER + ALIGN
> **Layer:** Structure (terminal/interaction model is a long-lived primitive)
> **Interview notes:** User confirmed this is about building deterministic control — the `work` skill FSM (currently a prompt) needs to become a FlowMind YAML so the lifecycle is baked into the Leviathan runtime.

## Raw Concepts

### The Problem
Need an abstraction for CLI coding agents (Codex, Claude Code) that:
- Still prints all output on screen
- Feels like talking to the arch while watching the dev team work
- Supports introspecting individual team members (named voices)
- Claude Teams: each member reports as another voice with names assigned

### Proposed Architecture
- **Terminal layer:** tmux for full control
- Message sent by user is hidden but still sent (one-way or filtered display)
- Some sort of abstraction over the terminal — implementation detail of the terminal
- This is the **interaction model for everything** — not just code

### Broader Vision
- Flows into the live dashboard concept
- Agent rendering UI, doing stuff on screen
- Could be video editing, research, etc.
- Duplex from NVIDIA mentioned as reference

### Relationship to Existing Code
- Relates to `core/harness/` — the agent runtime abstraction
- relates to `core/flowmind/` — the workflow execution engine
- Would live in `plugins/platforms/` or `core/harness/terminal/`

## Key Insight (from interview)
> "The work skill — I invoke manually. The dump is about what to build ON TOP of everything I have. Skills are great but we need deterministic control. The FSM — that's a prompt. It needs to be baked into Leviathan as a FlowMind YAML, and then we have full control over the lifecycle of everything."

## Open Questions
- [ ] What is the exact terminal abstraction? (tmux session management?)
- [ ] How does the agent/human message flow work in duplex mode?
- [ ] What's the relationship between the terminal abstraction and AgentPing action surfaces?

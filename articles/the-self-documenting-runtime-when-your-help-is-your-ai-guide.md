---
title: "The Self-Documenting Runtime: When Your --help IS Your AI Guide"
description: "How dev-browser's pattern of making --help output serve as the LLM guide represents the future of agent-native tool design. Auto-generating skill docs from contracts."
tags: [self-documenting, cli, dev-browser, flowmind, skills, agent-design]
date: 2026-04-10
series: leviathan-engineering
---

# The Self-Documenting Runtime: When Your --help IS Your AI Guide

The best documentation for an LLM is the one that cannot drift from the implementation.

## The Documentation Drift Problem

Every team that has built tools for AI agents has hit the same wall. You write a tool. You write docs for the tool. You write a SKILL.md or a system prompt that explains the tool to the agent. Then the tool changes, the docs lag behind, and the agent starts failing because its instructions no longer match reality.

This is not a discipline problem. It is a structural one. Whenever documentation lives in a separate artifact from the thing it documents, drift is inevitable. The question is not whether your docs will go stale, but when.

Sawyer Hood's dev-browser project demonstrated an elegant solution: make `--help` output serve double duty as both the human reference and the LLM guide. The runtime documents itself. There is no separate SKILL.md to maintain. The canonical description of what the tool can do is generated from the same code that implements the capability.

99% success rate in benchmarks. $0.078 per task. No documentation file to maintain.

## What Self-Documenting Actually Means

Self-documenting is an overused term, usually meaning "the code is readable." That is not what we mean here. We mean something specific and structural:

**The runtime's own output, when invoked with standard help flags, produces text that is sufficient for an LLM to use the tool correctly without any supplementary documentation.**

This has concrete requirements:

1. **Every subcommand describes its purpose in one sentence.** Not what it does technically, but what problem it solves.

2. **Arguments and flags include type hints and defaults.** An LLM reading `--timeout <ms> [default: 30000]` knows exactly what to pass.

3. **Examples are executable.** Not pseudocode, not descriptions of what you might do. Actual commands that work.

4. **Output format is described.** The agent needs to know what shape the response will take before it invokes the command.

5. **Error states are enumerated.** Not just "something went wrong" but specific error codes the agent can pattern-match on.

dev-browser achieves this through a persistent daemon architecture. The CLI communicates with a daemon that maintains page state, and every command's help text describes both the input it accepts and the output it produces. The agent never needs to consult a separate document.

## From Help Text to Skill Compiler

Leviathan takes this pattern further with a specific mechanism: FlowMind declarations compile into skill documentation automatically.

Here is the chain:

1. A module declares its capabilities in a `config.yaml` with FlowMind declarations -- the handlers in `src/handlers/` it exposes, the inputs they accept, the outputs they produce.

2. The FlowMind compiler reads these declarations and produces an execution contract -- a structured representation of what the module can do.

3. A skill projection takes that execution contract and generates prose documentation -- the equivalent of a SKILL.md, but derived from the source of truth rather than maintained separately.

4. That prose is what the agent sees when it asks "what can this tool do?"

The critical property: when a handler changes its signature, the skill documentation updates automatically on the next compilation pass. There is no separate artifact to forget to update. The documentation and the implementation share a single source of truth.

## The Three Layers of Self-Documentation

In practice, self-documenting runtimes operate at three layers:

### Layer 1: Static Help (Floor)

Standard `--help` output. Every subcommand, every flag, concise descriptions. This is what dev-browser does well and what AXI Principle 10 mandates. It is the minimum viable documentation for agent consumption.

### Layer 2: Contextual Disclosure (Standard)

After every response, the tool suggests next-step commands. "You just navigated to the login page. Next: `fill @username-field 'admin' --submit`." This is AXI Principle 9 in action. The tool does not just describe itself in the abstract -- it tells the agent what to do right now, given the current state.

### Layer 3: Contract-Derived Skills (Ceiling)

The full FlowMind compilation chain. Handler contracts, input/output schemas, validation gates, and example invocations all derived from the same YAML declarations that drive the runtime. This is where the documentation is not just up-to-date but provably correct -- it is generated from the same artifact that the runtime executes.

## Why This Matters for Agent Reliability

The benchmark data tells the story. Tools with self-documenting patterns consistently outperform tools that rely on separate documentation:

- **Fewer turns.** The agent does not need to probe the tool to understand it. The information is right there in the help output and contextual suggestions.

- **Lower cost.** Less exploration means fewer token-consuming round trips.

- **Higher success rate.** When the documentation matches the implementation, the agent's first attempt is more likely to be correct.

As agent systems scale, the number of tools an agent might use grows. Nobody maintains hand-written documentation for 80+ commands (the scale of tools like agent-browser). The only documentation strategy that scales is one where documentation generates from the code. Prompts converge, harnesses differentiate -- and the harness includes the documentation layer.

## The Daemon Pattern

dev-browser's architecture is worth examining because it solves a second problem: state management.

The CLI communicates with a persistent daemon over a Unix socket. The daemon maintains browser page state -- the current URL, the DOM snapshot, the element references. When the agent runs `click @submit-button`, the daemon knows which page it is looking at because it has been maintaining that context across invocations.

This maps directly to Leviathan's daemon lifecycle pattern. The CLI never runs business logic directly. The daemon owns all runtime state. This separation means the help text can describe operations in terms of the current state ("click an element on the current page") rather than requiring the agent to manage state externally ("open a browser, navigate to URL, find element, click element").

Stateful self-documentation is qualitatively different from stateless self-documentation. It does not just tell the agent what the tool can do -- it tells the agent what the tool can do right now, in this context.

## Building Self-Documenting Tools

If you are building tools that AI agents will use, here is the practical checklist:

**Start with the help text.** Before you implement a handler, write its `--help` output. What does it accept? What does it return? What errors can it produce? This is contract-first development applied to CLI design.

**Make every response actionable.** After every command output, include the 2-3 most likely next commands. This is cheap to implement and dramatically reduces agent turns.

**Generate, do not write.** If your tool has more than 10 commands, hand-maintaining documentation is a losing game. Build a generation step that derives docs from your handler contracts.

**Test with an LLM.** Give your help text to a model and ask it to complete a task. If it cannot figure out how to use your tool from the help text alone, the help text is insufficient.

**Treat documentation drift as a bug.** Not a low-priority cleanup task. A bug. If your documentation does not match your implementation, your tool is broken for every agent that reads it.

The future of agent-native tools is not better protocols or richer schemas. It is tools that explain themselves truthfully, in real time, from the same source of truth that drives their behavior. $0.074/task at 100% success was achieved partly because the tool's self-documentation eliminated exploration overhead -- 79K tokens/session vs 185K for the version with separate docs. The `--help` output is not an afterthought. It is the interface.

---

*This article is part of the Leviathan engineering series exploring the design principles behind the universal agent runtime. Leviathan is open source at [github.com/lev-os/lev](https://github.com/lev-os/lev).*

---
title: "Agent-Native CLI Design: Why MCP vs CLI Is the Wrong Debate"
description: "A deep dive into AXI's 10 principles for agent-ergonomic CLIs. The benchmarks that prove principled design beats protocol choice every time."
tags: [axi, cli, mcp, agent-design, constraint-engineering, benchmarks]
date: 2026-04-10
series: leviathan-engineering
---

# Agent-Native CLI Design: Why MCP vs CLI Is the Wrong Debate

The AI tooling world has spent the last year locked in a protocol war. MCP or CLI? Tool schemas or shell commands? Structured invocation or plain text?

490 benchmark runs and real production data say: it does not matter. What matters is design discipline -- 10 principles that consistently produce better outcomes regardless of which protocol carries them. Prompts converge, harnesses differentiate.

## The Misframed Debate

When teams evaluate how to expose functionality to AI agents, the conversation almost always starts with protocol selection. Should we build an MCP server? Should we wrap our CLI? Should we generate OpenAPI schemas?

Wrong first question. It is like debating REST vs gRPC before you have decided what your API contract looks like. The transport is a detail. The constraints are the product.

Kun Chen, a veteran of Meta, Microsoft, and Atlassian who now works on agent-native developer tools, formalized this insight into a framework called AXI: the Agent eXperience Interface. It is not a protocol. It is not a runtime. It is a design standard -- ten principles that make any interface agent-ergonomic, whether that interface speaks MCP, CLI, HTTP, or smoke signals.

## The 10 Principles

AXI organizes its principles into three categories: efficiency, robustness, and discoverability.

### Efficiency (Principles 1-3)

**1. Token-efficient output.** Every token an agent reads costs money and consumes context window. AXI promotes TOON (Token-Oriented Object Notation), which achieves roughly 40% savings over JSON for structured output. When your agent is making hundreds of tool calls per session, that savings compounds into real dollars and, more critically, into context window headroom that keeps the agent coherent longer.

**2. Minimal default schemas.** Show 3-4 fields per list item by default. Provide `--fields` for expansion. The default output should contain exactly what an agent needs for the next decision, nothing more. This is the opposite of the "dump everything and let the LLM figure it out" approach, which wastes tokens and degrades accuracy.

**3. Content truncation with size hints.** Large outputs should be truncated by default with explicit size hints. A `--full` flag provides the escape hatch. Agents can decide whether they need the full payload based on the size hint, rather than being forced to ingest it all.

### Robustness (Principles 4-6)

**4. Pre-computed aggregates.** A single `fill @uid text --submit` command that fills a field, submits the form, waits for navigation, and returns the new page snapshot. One round trip instead of four. Combined operations reduce the number of turns an agent needs, which directly reduces cost and latency.

**5. Definitive empty states.** When there are no results, say "0 results." Never return silence. An LLM interpreting an empty response will hallucinate explanations for why nothing came back. An explicit empty state kills that ambiguity dead.

**6. Structured errors on stdout.** Errors go to stdout, not stderr. They are structured, not free-text. Mutations are idempotent. And critically: no interactive prompts, ever. An agent cannot answer "Are you sure? [y/n]". Any tool that asks will fail 100% of the time in an autonomous context.

### Discoverability (Principles 7-10)

**7. Ambient context via session hooks.** The tool self-installs into the agent's session lifecycle. When the session starts, the tool provides context about its current state. This eliminates the "cold start" problem where an agent has to probe for what tools are available and what state they are in.

**8. Content first.** Running a command with no arguments should return live data, not help text. `git status` understood this decades ago. `lev work` should show current workstream status, not a usage message. The no-args invocation is the most common agent invocation, and it should be the most useful one.

**9. Contextual disclosure.** After every response, include next-step command templates. Do not make the agent guess what it can do next. Show it. "You just listed 3 open specs. To start working on one: `lev work start spec-agentfs`." This is progressive disclosure tuned for agents.

**10. Consistent `--help`.** Every subcommand has a concise, consistent help output as a fallback. This is the floor, not the ceiling.

## The Benchmarks That Settle It

Theory is cheap. Here is what 490 benchmark runs across four browser automation tools showed:

| Tool | Success Rate | Cost/Task | Avg Duration | Avg Turns |
|---|---|---|---|---|
| chrome-devtools-axi | **100%** | **$0.074** | **21.5s** | **4.5** |
| dev-browser | 99% | $0.078 | 28.6s | 4.9 |
| agent-browser | 99% | $0.088 | 24.6s | 4.8 |
| chrome-devtools-mcp | 99% | $0.101 | 26.0s | 6.2 |

The same underlying capability (Chrome DevTools Protocol) exposed through AXI principles versus MCP tool schemas. Same Chrome. Same tasks. Same model.

The AXI-principled CLI hit 100% success at the lowest cost. The MCP version of the same tool costs 36% more per task and requires 38% more turns. The difference is not the protocol. The difference is that MCP tool schemas consume 28.5% of input tokens just to describe themselves. The AXI version averages 79K tokens per session versus MCP's 185K.

This is not a marginal difference. It is the difference between an agent that works reliably and one that starts dropping context after a few interactions.

## Why This Maps to Constraint Engineering

In Leviathan's architecture, these are not abstract guidelines. They map directly to the system's root constraints.

Token-efficient output is a direct consequence of C1 Finitude: all resources are finite. Time, tokens, context, memory, compute, attention -- every operation must be bounded. If your tool output wastes tokens, you are violating the most fundamental constraint in the system. Gates = loss function. Waste is measurable, and the gate catches it.

Session hooks map to Leviathan's reactive hooks system (`~/.config/lev/reactive/hooks.yaml`), which already provides lifecycle hooks for format, lint, and fix pipelines. Structured errors map to LevEvent-only semantics -- the canonical event schema that all inter-module communication must use.

The content-first principle maps to Leviathan's handler-first design philosophy. The `src/handlers/` convention (never `src/commands/` -- that name implies CLI and causes surface overloading) exists precisely because business logic should be surface-agnostic. A handler that produces content-first output works the same whether invoked via CLI, MCP, HTTP, or gRPC. A handler designed around a specific protocol's affordances locks you into that protocol.

## The Practical Implication

If you are building tools for AI agents today, stop debating protocols and start applying these principles:

1. **Audit your token budget.** Measure how many tokens your tool's output consumes per invocation. If it is more than what the agent needs for its next decision, you are wasting money and context.

2. **Kill all interactive prompts.** Grep your codebase for anything that expects stdin. Replace it with flags, defaults, or structured error responses.

3. **Make no-args useful.** If your tool shows help text when invoked without arguments, change it to show current state instead.

4. **Add next-step templates.** After every response, include the 2-3 most likely next commands the agent will want to run.

5. **Measure, do not guess.** Run your tool through a benchmark harness with an actual LLM. Count turns, cost, and success rate. The numbers will tell you where your design is failing.

The MCP vs CLI debate will continue. People enjoy protocol arguments. But the teams that ship reliable agent-native tools will be the ones who internalized these design principles regardless of which wire format they chose.

Principled design beats protocol choice. 490 benchmark runs across 4 tools prove it. $0.074/task at 100% success vs $0.101/task at 99% -- same Chrome, same tasks, same model, different design discipline. The constraint framework explains why.

---

*This article is part of the Leviathan engineering series exploring the design principles behind the universal agent runtime. Leviathan is open source at [github.com/lev-os/lev](https://github.com/lev-os/lev).*

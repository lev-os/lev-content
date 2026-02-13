---
title: Core Concepts
description: Shared vocabulary for FlowMind, Ratchet, AgentPing, and runtime behavior.
order: 30
section: concepts
---

# Core Concepts

## FlowMind

FlowMind is the runtime declaration/execution substrate.

- declares triggers, routing, and execution steps
- enforces deterministic structure around agent actions
- provides a consistent way to model lifecycle transitions

## Ratchet

Ratchet is fail-closed constraint enforcement.

- policies tighten over time instead of drifting open
- schema and boundary checks happen before action execution
- unknown/invalid actions are denied by default

## AgentPing

AgentPing is the agent-to-human action surface.

- typed interaction requests instead of ad hoc chat
- explicit responses: approve, reject, select, ask
- built for async, multi-surface workflows

## NTM / Background Orchestration

NTM-style orchestration is headless agent execution control.

- spawn/manage workers
- inspect status and output
- keep orchestration explicit and observable

## Timetravel and Tracing

- Timetravel: research adapter strategy surface (multi-source retrieval)
- Tracing: inspect and replay decision/execution paths for debugging

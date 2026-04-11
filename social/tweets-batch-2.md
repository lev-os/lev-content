# Tweets Batch 2 — @levosdev

---

### 1 (Comparison)

Most agent frameworks let you chain LLM calls and call it orchestration.

Lev starts from two axioms: all resources are finite (C1) and order matters (C2). Everything else — gates, receipts, bounded loops — derives from those constraints.

You don't need more abstractions. You need fewer, better ones.

---

### 2 (Comparison)

CrewAI: "define agents and tasks"
LangGraph: "define nodes and edges"
Lev: "define constraints, then let the runtime prove your agent stayed inside them"

The difference is who checks the homework. In Lev, the agent never marks its own.

---

### 3 (Comparison)

Every framework gives you `run()`. Lev gives you a tick:

INGEST -> OBSERVE -> PROPOSE -> GATE -> APPLY -> UPDATE -> EMIT

Every message passes through gates. Every mutation is a proposal that can be rejected. That's not overhead — that's the whole point.

---

### 4 (Comparison)

We benchmarked 3 browser automation strategies: dev-browser (Playwright), agent-browser (cloud), and chrome-devtools-axi (accessibility tree).

AXI won. Not close. Smaller context, faster execution, more reliable selectors.

The cascade router falls through automatically if your preferred engine fails. One interface, many backends.

---

### 5 (Comparison)

"SDK-first" means your business logic doesn't know whether it's being called from a CLI, an MCP server, or an HTTP endpoint.

`core/poly/src/surfaces/cli/` and `core/poly/src/surfaces/mcp/` are thin projections of the same registry. No `src/commands/` folder. No surface-coupled code.

Most frameworks are CLI-first and bolt on APIs later. We did it the other way.

---

### 6 (How-to)

Quick start with Lev's iterative loop:

```
lev exec --until "all tests pass" --max-turns 10
```

The runtime calls your provider, checks the exit condition semantically after each turn, detects idle drift, and stops when done or budget-exhausted. Git commits between iterations. That's the Ralph loop.

---

### 7 (How-to)

Want your agent to browse the web? Lev's browser router supports 5 engines:

- DevBrowser (Playwright)
- AgentBrowser (cloud)
- BrowserUse
- PinchTab
- Skyvern

Set `engines: [axi, dev-browser]` in your flow YAML. The cascade handles fallback automatically.

---

### 8 (How-to)

Define system constraints before writing code:

```yaml
# dna/graph.yaml
constraints:
  C1_finitude:
    axiom: "All resources are finite."
    violation: FATAL
```

Your DNA files are loaded by the runtime. Gates enforce them at proposal time. Not documentation — executable contracts.

---

### 9 (How-to)

AgentFS gives your agent a sandboxed filesystem via FUSE. Copy-on-write, audit trails, hook runners on every operation.

The agent thinks it's writing to disk. It's actually writing to a controlled, inspectable, rollback-capable layer. Written in Rust. Ships as a single binary.

---

### 10 (How-to)

Lev's `--help` output IS the LLM system prompt. Not a separate doc site. Not a stale README.

When an agent calls `lev --help`, it gets the same contract a human reads. Self-documenting runtime means the docs can't drift from the code.

---

### 11 (Philosophy)

Hot take: the agent framework wars are solving the wrong problem.

The hard part isn't chaining LLM calls. It's preventing an autonomous system from silently drifting into an invalid state while you're not watching.

Constraints first. Capabilities second.

---

### 12 (Philosophy)

"Local-first, federate-ready. Must work on a Raspberry Pi. Must scale to a galactic fleet."

That's not a joke. It's a design constraint. If your agent runtime requires a cloud account to function, you've already lost the portability game.

---

### 13 (Philosophy)

Every agent framework eventually reinvents the filesystem, the event bus, and the permission model.

We decided to just... build those properly from the start. AgentFS (FUSE), LevEvent bus (append-only), AgentGuard (scoped, time-bounded permissions).

---

### 14 (Philosophy)

"YAML is DNA. These files define the organism. The runtime expresses them."

Your system's behavior should be auditable by reading YAML, not by tracing execution paths through 40 files of framework code. Declarations over implementations.

---

### 15 (Philosophy)

Receipts are immutable completion proofs. Content hash, eval results, effect results, provenance — all bundled into one artifact.

If your agent says "done" but can't produce a receipt, it isn't done. This is the finality boundary. We think every framework needs one.

---

### 16 (Community)

Question for the agent builders out there:

What's the worst thing an autonomous agent has done in your codebase when you weren't looking?

We designed Lev's gate system specifically because we kept getting burned. Curious what patterns others have seen.

---

### 17 (Community)

Poll idea: when your agent loops on a task, how do you detect it's stuck?

a) Token count heuristics
b) Semantic similarity between iterations
c) Wall-clock timeout
d) You don't, and it burns $40

Lev does (a) + (b) automatically via the idle detector in the iterative runner. We think (d) is too common.

---

### 18 (Community)

We're building 7 domain routers in the open: Storage, Memory, Browser, AgentFS, Reactive, DNA, Eval.

Which domain plugin would be most useful to your workflow? Drop a reply — we're prioritizing based on real demand, not roadmap theater.

---

### 19 (Community)

If you've ever tried to make an LLM reliably fill out a form on a website, you know the pain.

We wrote a trajectory recorder that captures every browser step as structured data. Considering open-sourcing the dataset format separately. Would that be useful to anyone?

---

### 20 (Community)

We use the term "constraint engineering" instead of "prompt engineering."

Prompts tell the model what to do. Constraints tell the runtime what the model is NOT allowed to do. Different discipline. Different failure modes.

Is anyone else thinking about agent development this way? Genuinely want to find the others.

---

### 21 (Builder)

Shipped this week: entity lifecycle loop prompt with case-study-driven routing. The system picks execution strategies based on prior successful patterns, not hardcoded if/else trees.

Building in public means you can read the commit: `4eab1405e9`.

---

### 22 (Builder)

Current codebase topology:

- 18 core modules (harness, exec, flowmind, orchestration, poly, daemon...)
- 12+ plugins (browser, auth-sniffer, evolve-memory, sdlc...)
- AgentFS in Rust (FUSE mount, CoW, hook runners)
- 61 specs with validation gates cross-referenced in YAML

All open. All contract-first.

---

### 23 (Builder)

We deleted 5,000+ lines of dead code over 117 chore ticks across 12 sessions. Wrote 150+ tests to backfill.

Maintenance isn't the opposite of shipping. It's what makes shipping sustainable. The autodev loop pattern made this tractable.

---

### 24 (Builder)

The execution contract is a compiled state machine IR. FlowMind declares it, Orchestration executes it. Pure functions in, structured results out.

This separation is why the same workflow definition can run in a CLI, a daemon, a browser WASM sandbox, or a mobile webview. Seven hosts, one semantic runtime.

---

### 25 (Builder)

The hardest part of building Lev isn't the code. It's resisting the urge to ship features before the constraints are defined.

Every time we skip the DNA file and go straight to implementation, we pay for it in the next three sessions. Every time.

Constraints first. Always.

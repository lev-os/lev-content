# Motia Runtime Comparison

> **Source:** `00-vision.md` lines 354-2635 (raw HTML dump from motia.dev/docs/concepts/workbench)
> **Status:** captured → needs analysis via `lev-intake` flow
> **Layer:** Services (runtime adapter evaluation)
> **Interview notes:** User says Motia is a runtime adapter candidate. May be superseded, but wants to set it up and see what happens. Potential home: `plugins/platforms/motia`. They've spent time refining their software, so worth evaluating.

## What Motia Is

An event-driven backend runtime with a visual development workbench. Key primitives:

| Motia Concept | Lev Equivalent | Notes |
|--------------|---------------|-------|
| **Steps** | Historical mapping: FlowMind workflows. Current: source counterpart only; exact legacy Motia/iii runtime equivalence is not established. | Historical description: Event-driven DAG nodes. Types: API, Event, Cron<br>Current owner audit 2026-10-16: [unchanged] FlowMind compiles authored documents for supported targets. Current Motia URL resolves to successor iii with Workers/Functions/Triggers; the historical API/Event/Cron Steps contract is not assumed unchanged. Lev: `core/flowmind/src/compiler.ts:109-145`. Upstream: [iii successor introduction](https://github.com/iii-hq/iii/blob/05440e1fb1b5080285c156ab6fd1a4def4310ae2/README.md#L42-L83). Source only; runtime parity unaccepted. |
| **Workbench** | Historical mapping: Dashboard vision (00e). Current: source counterpart only; exact legacy Motia/iii runtime equivalence is not established. | Historical description: Visual flow view, API testing, tracing, state inspection<br>Current owner audit 2026-10-16: [unchanged] AgentPing supplies a real routed web app. Its route declarations do not establish a Workbench with matching API tests, trace timeline or state inspection; iii composition/observation is not legacy Workbench compatibility. Lev: `community/agentping/packages/adapters/web-ui/src/App.tsx:17-43`. Upstream: [iii successor introduction](https://github.com/iii-hq/iii/blob/05440e1fb1b5080285c156ab6fd1a4def4310ae2/README.md#L42-L83). Source only; runtime parity unaccepted. |
| **Event Bus** | Historical mapping: `core/bus/`. Current: source counterpart only; exact legacy Motia/iii runtime equivalence is not established. | Historical description: Pub/sub event routing between steps<br>Current owner audit 2026-10-16: [unchanged] Lev emits events to matching subscribers and returns unsubscribe handles. This is actual pub/sub source, not merely envelope normalization; transport/durability and legacy Motia delivery equivalence remain separate. Lev: `core/event-bus/src/events/event-bus.ts:128-197`. Upstream: [iii successor introduction](https://github.com/iii-hq/iii/blob/05440e1fb1b5080285c156ab6fd1a4def4310ae2/README.md#L42-L83). Source only; runtime parity unaccepted. |
| **State** | Historical mapping: Beads / BD. Current: source counterpart only; exact legacy Motia/iii runtime equivalence is not established. | Historical description: `state.set()` / `state.get()` — persistent key-value<br>Current owner audit 2026-10-16: [unchanged] The typed memory adapter POSTs MemoryItem fields to its configured SQLite service. This is not evidence for a general state.set/state.get contract; Beads is not the universal runtime KV owner. The iii introduction does not certify the historical Motia KV API. Lev: `core/memory/src/backends/sqlite.ts:95-118`. Upstream: [iii successor introduction](https://github.com/iii-hq/iii/blob/05440e1fb1b5080285c156ab6fd1a4def4310ae2/README.md#L42-L83). Source only; runtime parity unaccepted. |
| **Tracing** | Historical mapping: Agent tracing (planned). Current: source counterpart only; exact legacy Motia/iii runtime equivalence is not established. | Historical description: Trace ID → execution timeline → step breakdown<br>Current owner audit 2026-10-16: [closer] Lev withSpan starts a named span, records callback status/errors and closes it when telemetry is enabled; it falls back to the callback when disabled. This narrows the historical planned-only label, but does not prove matching Workbench trace/state timelines. Lev: `core/telemetry/src/spans.ts:65-108`. Upstream: [iii successor introduction](https://github.com/iii-hq/iii/blob/05440e1fb1b5080285c156ab6fd1a4def4310ae2/README.md#L42-L83). Source only; runtime parity unaccepted. |
| **Hot Reload** | Historical mapping: `core/lifecycle/` hot reload. Current: source counterpart only; exact legacy Motia/iii runtime equivalence is not established. | Historical description: File change → auto-refresh, no restart<br>Current owner audit 2026-10-16: [unchanged] The actual bounded hot-reload/watcher-reload scan found no match in FlowMind, Exec and event-bus source. That is not repository-wide absence proof. Neither a lifecycle enum nor graph execution establishes watcher-driven hot reload; the current iii introduction does not certify the legacy behavior. Lev: `rg -n -i -g '*.ts' -g '*.tsx' -e 'hot.?reload\|watch.*reload\|reload.*watch' core/flowmind/src core/exec/src core/event-bus/src` (exit 1, 0 matches in named owners). Upstream: [iii successor introduction](https://github.com/iii-hq/iii/blob/05440e1fb1b5080285c156ab6fd1a4def4310ae2/README.md#L42-L83). Source only; runtime parity unaccepted. |
| **Cron Steps** | Historical mapping: The Tick (00d). Current: source counterpart only; exact legacy Motia/iii runtime equivalence is not established. | Historical description: Scheduled tasks on a timer<br>Current owner audit 2026-10-16: [unchanged] Lev has schedule occurrence/tick source. iii Triggers document cron schedules; this does not prove identical legacy Motia Cron Step lifecycle or an installed timer run. Lev: `plugins/scheduling/src/cron-tick-provider.ts:352-390`. Upstream: [iii successor introduction](https://github.com/iii-hq/iii/blob/05440e1fb1b5080285c156ab6fd1a4def4310ae2/README.md#L42-L83). Source only; runtime parity unaccepted. |
| **Polyglot** | Historical mapping: `core/polyglot-runners/`. Current: source counterpart only; exact legacy Motia/iii runtime equivalence is not established. | Historical description: TS, JS, Python, Ruby steps in same project<br>Current owner audit 2026-10-16: [unchanged] Lev command binding metadata and registry resolution are concrete. The current iii introduction gives TypeScript/Python/Rust worker examples; retain historical Motia TS/JS/Python/Ruby as history, not a tested language-by-step runner matrix. Lev: `core/poly/src/registry/commands.ts:312-341`. Upstream: [iii successor introduction](https://github.com/iii-hq/iii/blob/05440e1fb1b5080285c156ab6fd1a4def4310ae2/README.md#L42-L83). Source only; runtime parity unaccepted. |

## What's Interesting

1. **Workbench UI** — closest existing implementation to the dashboard vision
   - Flow view (interactive graph of backend)
   - Endpoint view (built-in API testing — no Postman needed)
   - Debug panel (tracing timeline, real-time logs, state inspector + editor)
   - Notification routing with trace IDs

2. **Step architecture** — clean event-driven DAG with types
   - API Step (entry point, green nodes)
   - Event Step (background job, blue nodes)
   - Cron Step (scheduled, orange nodes)
   - Connections show data flow

3. **State management** — read/write state from any step, inspect in workbench

## User Intent

> "Motia is a runtime adapter. I want to set it up and have compatibility with Leviathan — `plugins/platforms/` or something. They already have everything working. We have a week or so before I can see anything — I want to get some Leviathan primitives sorted and then hook it up to Motia and see what happens. It would be a nice dashboard for introspection."

## Action Items
- [ ] Clone Motia and set up locally (via `lev-intake`)
- [ ] Evaluate as `plugins/platforms/motia` adapter
- [ ] Map Motia Steps to FlowMind YAML format
- [ ] Evaluate Workbench as dashboard starting point
- [ ] Determine: superseded or complementary?

## Raw Source
The original HTML dump (2281 lines of raw Motia docs HTML) has been removed from this file. The original content came from `https://motia.dev/docs/concepts/workbench` and can be re-fetched via `lev-intake` if needed.

# Motia Runtime Comparison

> **Source:** `00-vision.md` lines 354-2635 (raw HTML dump from motia.dev/docs/concepts/workbench)
> **Status:** captured → needs analysis via `lev-intake` flow
> **Layer:** Services (runtime adapter evaluation)
> **Interview notes:** User says Motia is a runtime adapter candidate. May be superseded, but wants to set it up and see what happens. Potential home: `plugins/platforms/motia`. They've spent time refining their software, so worth evaluating.

## What Motia Is

An event-driven backend runtime with a visual development workbench. Key primitives:

| Motia Concept | Lev Equivalent | Notes |
|--------------|---------------|-------|
| **Steps** | FlowMind workflows | Event-driven DAG nodes. Types: API, Event, Cron |
| **Workbench** | Dashboard vision (00e) | Visual flow view, API testing, tracing, state inspection |
| **Event Bus** | `core/bus/` | Pub/sub event routing between steps |
| **State** | Beads / BD | `state.set()` / `state.get()` — persistent key-value |
| **Tracing** | Agent tracing (planned) | Trace ID → execution timeline → step breakdown |
| **Hot Reload** | `core/lifecycle/` hot reload | File change → auto-refresh, no restart |
| **Cron Steps** | The Tick (00d) | Scheduled tasks on a timer |
| **Polyglot** | `core/polyglot-runners/` | TS, JS, Python, Ruby steps in same project |

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

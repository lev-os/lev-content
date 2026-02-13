# Vernacular (LOCKED)

> Extracted from `docs/01-architecture.md`. These terms are canonical. Use them consistently across all docs, code, and conversation.

---

## Core Terms (5)

| Term | Meaning | Location | Example |
|------|---------|----------|---------|
| **Harness** | Hex execution engine (domain core) | `harness/` | Orchestrates prompt execution |
| **Provider** | LLM execution backend | `harness/providers/` | ClaudeSDKProvider, ShellProvider |
| **Platform** | External system we integrate with (IDE, gateway, agent CLI) | Canonical: `plugins/platforms/src/<name>/` | cursor/, claude-code/, openclaw/ |
| **Runner** | Build-time registry unit | `polyglot-runners/` | Binary, daemon, SDK command |

---

## Reactive Layer Terms (7 — LOCKED, Session 4, 2026-01-31)

Extends the 5-term vernacular above. These terms replace all prior usage of "hook" (overloaded 5 ways), "signal", "sensor", "reactor".

| Term | Meaning | Example |
|------|---------|---------|
| **Event** | Normalized JSON payload on the bus — the LevEvent | `{ version: 1, type: "git.commit.post", source: "git", ts: "2026-01-31T00:00:00Z", time: "2026-01-31T00:00:00Z", id: "abc", data: {...} }` |
| **Source** | Field on Event identifying origin system | `git`, `cron`, `fs`, `imsg`, `bd`, `session`, `xstate`, `tool`, `conversation` |
| **EventProvider** | Translates native source events into LevEvent format (sources WITH native events) | GitEventProvider, CronEventProvider, ImsgEventProvider, SessionEventProvider, BdEventProvider, XStateEventProvider |
| **Watcher** | Polls sources WITHOUT native events, emits LevEvent | FileWatcher (fs), TranscriptWatcher (conversation) |
| **Trigger** | ⚠️ COLLISION — needs FlowMind deep dive. Currently 3 meanings: triggers/ module rules, FlowMind `trigger:` field, lifecycle patterns | TBD after FlowMind vernacular |
| **Action** | What executes in response — FlowMind dispatch | FlowMind workflow, shell cmd, BD operation |
| **Bus** | Central pub/sub event router | events/EventBus (canonical spine) |
| **EventLog** | Durable JSONL persistence + replay | events/JsonlPersistence → `~/.local/share/lev/events.jsonl` |

### EventProvider vs Watcher (smoke-tested with 3 models — unanimous)

- **EventProvider** = source fires events natively (git hooks, cron ticks, imsg watch, XState transitions) — we translate to LevEvent
- **Watcher** = source has no native events (filesystem, conversation transcripts) — we poll and emit LevEvent
- Both emit to Bus. Different mechanics, same output.

---

## FlowMind Outputs (Polymorphic)

FlowMind outputs are polymorphic — not a single "Flow" artifact:

- System prompt (smartdown target)
- Action hook (on event → run steps)
- Runtime context injection (harness injects mid-execution)
- Bidirectional whisper (agent ↔ workflow)
- `.flow.yaml` = the spec file, FlowMind = the compiler, output = depends on target

---

## Reactive Pipeline

```
Native sources (git, cron, imsg, bd, session, XState, tool, timetravel)
    → EventProvider translates → LevEvent → Bus

Polled sources (fs, conversation)
    → Watcher polls → LevEvent → Bus

All Events on Bus:
    → Trigger matches pattern (⚠️ needs FlowMind unification)
    → Action dispatches
    → FlowMind outputs: system prompt / hook execution / runtime injection / whisper
    → EventLog persists for replay
```

---

## Interaction Layer Terms (LOCKED, Session vision-00b, 2026-02-12)

Extends the reactive layer. These terms resolve the AgentPing/AgentLease/AgentGate vernacular.

| Term | Meaning | Brand Name | Example |
|------|---------|-----------|---------|
| **ActionSurface** | Discrete intent→action boundary — where interaction happens | AgentPing | Browser button, CLI command, stream deck, voice, SMS, VR gesture |
| **Interface** | Continuous interaction medium over an action surface | — | Conversation, VR presence, dashboard session |
| **Gate** | Evaluator that allows/amends/rejects proposals (enforces leases) | AgentGate | Gate/Scope, Gate/Safety, Gate/Standards |
| **Lease** | Rule/contract an agent must fulfill before proceeding | AgentLease | "Must pass tests before commit", "Must get human approval for deploy" |
| **ActionExtractor** | Converts Interface streams → ActionSurface events | — | Confidence thresholds, confirmations |

### Hierarchy

```
ActionSurface (where) → Interface (how) → Gate (checkpoint) → Lease (what's enforced)
```

### System Boundary

| System | Owns |
|--------|------|
| **LEV** | orchestration runtime, lifecycle graph, policy evaluation, worker execution |
| **AgentPing** | human-loop protocol, component kit, dashboard surfaces, approval UX |
| **AgentLease** | lease issuance, step-up auth, scoped permission grants |

---

## Lifecycle States (LOCKED, Session vision-04, 2026-02-12)

```
ephemeral → researching → crystallizing → committed → planning → executing → complete
```

| State | Command | Meaning |
|-------|---------|--------|
| ephemeral | — | Raw capture, stream of consciousness |
| researching | `/research` | Context gathering |
| crystallizing | `/work design` | Structure from chaos |
| committed | `/work spec` | Approved spec |
| planning | `/work plan` | Sequencing tasks |
| executing | `lev exec` / `bd` | Running tasks |
| complete | — | Done |

---

## CMS Adapter Model

The graph lives in a **CMS adapter** — filesystem is a view/closest truth match.

| Adapter | Type | Notes |
|---------|------|-------|
| **Beads (BD)** | Primary | Custom content types, current default |
| **BR** | Alternative | Similar capabilities |
| **TD** | Alternative | Similar capabilities |
| **WordPress** | Alternative | If supports custom content types |
| **SQLite direct** | Alternative | Lightweight local |
| **Notion** | Alternative | Cloud-native |
| **Obsidian** | Alternative | Markdown-native |

Adapters MUST NOT be hardcoded. Core operates against an abstract CMS interface.

**Note**: Raw types and enums don't need full BD treatment. Beads is one data source — future architecture supports multiple (GraphQL server, SQLite direct, headless CMS). The adapter interface is what matters.

---

## View Rendering Pipeline (LOCKED, Session 2026-02-12)

Documents are a **TYPE** (graph entities with classification). Not just views. `lev-agentfs` provides a way to reject misaligned docs and steer agents writing to the virtual file system.

The rendering pipeline:

```
BD hook adapter → LevEvents → template engine → structured markdown view
                                                  ↓
                                    .lev/pm/reports/{date}-{time}-{slug}.md
                                    .lev/pm/specs/{date}-{time}-{slug}.md
                                    .lev/pm/proposals/{date}-{time}-{slug}.md
```

- Agents get easy filesystem access to everything via `lev find` (graph search)
- Web API can serve the same views
- Templates enforce classification and structure
- AgentFS rejects writes that don't match entity type constraints

## Known Collisions (to resolve in FlowMind deep dive)

- "Provider" has 3 meanings: LLM backend (harness), EventProvider (reactive), Search backend (polyglot)
- "Trigger" has 3 meanings: triggers/ module, FlowMind `trigger:` field, lifecycle patterns
- "Event" has 3 historical variants: LifecycleEvent (observe), agent-adapter event schema (retiring), Event<T> (types) — consolidate to LevEvent

---

## Retired Terms

- ~~hook~~ → use Watcher (detection), EventProvider (translation), or Action (response)
- ~~signal~~ → use Event
- ~~sensor~~ → use Watcher
- ~~reactor~~ → use Action
- ~~Legacy event schemas~~ → use LevEvent (lightweight, lev-specific JSON schema)
- ~~Flow~~ → FlowMind outputs are polymorphic — no single artifact
- ~~Adapter for events~~ → use EventProvider (Adapter already means harness execution adapters)

---

## Approved Renames

- `agent-harness/` → `harness/` ✅ (task lev-qo1m.9)
- `harness/src/adapters/` → `harness/src/providers/` ✅ (class rename *Adapter → *Provider)
- `observe/` → `events/` ✅
- `agent-adapter/src/adapters/` → `plugins/platforms/src/` ✅ hard cut complete (2026-02-09)
- `integrations/` → `plugins/platforms/` ✅ canonical location
- `triggers/` → decomposed into flowmind/ + plugins/ (Session 5)
- `lib/fractal/` → `core/fractal/` (completed)
- `lib/middleware/` → `_archive/` (dead code)
- `lib/watch/` → `_archive/` (dead code)
- `lib/events/` → `_archive/` (stub, no package.json)
- Drop terms: "Adapter" (overloaded with hex pattern), "Client" (misleading) — replaced by "Integration" + "Platform"

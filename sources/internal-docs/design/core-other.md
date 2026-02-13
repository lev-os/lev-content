# 08 - Core: Plugins, Config & Other

**Status**: 🟡 RECONCILED (ownership + compliance snapshot)
**Epic**: lev-qo1m (Final Boss)
**Last Updated**: 2026-02-13

---

## Scope

**What this covers:**
- Plugin architecture (ships-with flag)
- Config system (XDG fractal)
- Domain types (nucleus)
- CLI routing
- Daemon ownership boundary and process supervision ownership

**Modules:**
```
~/lev/core/
├── plugins/            # Extension layer (ships-with flag)
│   └── ... see structure below
├── config/             # XDG fractal config
├── domain/             # Shared types (nucleus)
├── cli/                # lev command router
├── polyglot-runners/   # Poly binder + process supervision owner
├── daemons/            # Daemon business-logic services (runtime owner)
└── build/              # lev-dev tooling
```

---

## PLUGINS ARCHITECTURE (LOCKED)

### Structure
```
plugins/                                    # Extension layer
├── platforms/src/                          # IDE/gateway (ships-with: true)
│   ├── config-generator.ts                 # shared base
│   ├── transcript-watcher.ts               # shared base
│   ├── bridge.ts                           # shared base
│   ├── cursor/
│   ├── claude-code/
│   └── openclaw/
│
├── sdlc/src/                               # SDLC EventProviders (ships-with: true)
│   └── eventproviders/
│       └── git.ts                          # GitEventProvider
│
├── scheduling/src/                         # Time EventProviders (ships-with: true)
│   └── eventproviders/
│       └── cron.ts                         # CronEventProvider
│
├── timetravel/src/                         # ships-with: true
├── workshop/src/                           # ships-with: true
├── automem/src/                            # ships-with: false (optional)
└── graphiti/src/                           # ships-with: false (optional)
```

### ships-with Flag
```yaml
# plugins/sdlc/config.yaml
ships-with: true    # Included in default lev install
```

### Plugin Surface Types
- `config-generator` — emit config so lev works in platform environment
- `transcript-watcher` — scan conversation logs, emit LevEvents
- `bridge` — gateway ↔ harness middleware (platforms with gateways only)
- `eventproviders/` — translate native events → LevEvent

### Plugin Runtime Surface Contract (Current)
- `package.json` `leviathan` metadata describes plugin capabilities.
- Runnable surfaces (`cli`, `daemon`, `mcp`, command bindings) must be declared in `config.yaml` via `poly:` for lifecycle/registry ownership.
- Current gap: only `plugins/deploy/config.yaml` declares `poly.cli`; several plugins still ship bespoke runtime surfaces outside poly declarations.

Known bypasses (2026-02-13):
- `plugins/timetravel/` (`timetravel` CLI + scheduler daemon logic)
- `plugins/workflow-orchestrator/` (CLI + MCP adapter)
- `plugins/gemini-executor/` (CLI + MCP metadata)

---

## CONFIG SYSTEM (LOCKED)

### XDG Compliance
```
~/.config/lev/          # XDG_CONFIG_HOME/lev
~/.local/share/lev/     # XDG_DATA_HOME/lev
~/.local/state/lev/     # XDG_STATE_HOME/lev
~/.cache/lev/           # XDG_CACHE_HOME/lev
```

**NO ~/.lev pollution** — Legacy paths removed.

### Fractal Resolution
```
1. System:     ~/.config/lev/config.yaml
2. Project:    <project>/.lev/config.yaml
3. Module:     <module>/config.yaml
4. Environment: LEV_* env vars
```

Last wins. Deep merge.

---

## DOMAIN (NUCLEUS) — LOCKED

### What Goes Here
- `LevEvent` schema (canonical event type)
- Shared interfaces (`IEventBus`, `IProvider`, etc.)
- Value objects and domain primitives

### What Moved Here
- `core/types/` → merged into `core/domain/`
- `PACKAGE_SCHEMA.md` → `core/domain/`

### What Stays Elsewhere
- `context/schema/` stays as `@lev-os/schema` (active, has consumers)
- Module-local schemas stay local (FlowMind IR in flowmind/)

---

## DAEMON (RECONCILED)

**SRP target**:
- `core/polyglot-runners/`: process supervision and daemon lifecycle orchestration.
- `core/daemons/`: daemon application/business logic ownership.

**Current code reality**:
- legacy `core/daemon/` module still contains PID/heartbeat/signal/process lifecycle code and subsystem spawning.
- `core/polyglot-runners/` already implements PMDaemon-backed lifecycle adapters and daemon orchestration.
- `core/daemons/` still contains legacy daemon implementations (`lev-learner`, `parallel-research`).

**Convergence rule**:
- New or migrated daemons declare lifecycle intent in `config.yaml` (`poly.daemon`) and do not own bespoke lifecycle loops.

---

## ARCHIVED MODULES

```
~/lev/_archive/
├── lib-middleware/     # Dead code (0 imports)
├── lib-watch/          # Dead code (0 imports)
├── lib-events/         # Stub (no package.json)
├── cdo/                # Concepts only, no code
└── adapters/           # Unknown purpose, archived
```

---

## LOCKED DECISIONS

### Q1: workshop/ purpose ✅
**Decision**: CDO collaboration patterns (ships-with: true plugin)

### Q2: plugins/ location ✅
**Decision**: `~/lev/plugins/` at repo root (not core/plugins/)
**Reason**: Extension layer, not core infrastructure

### Q3: tools/ ✅
**Decision**: NO core/tools/ — tools live in plugins/ or harness/

### Q4: config/ ✅
**Decision**: XDG fractal system in `core/config/`

### Q5: cli/ ✅
**Decision**: `core/cli/` for lev command router

### Q6: adapters/ ✅
**Decision**: ARCHIVED — unknown purpose, dead code

---

## Epics

- lev-qo1m — Final Boss (architecture review parent)
- lev-qo1m.10 — Restructure plugins/ + move integrations/
- lev-qo1m.11 — triggers/ decomposition
- lev-qo1m.12 — lib/* audit + archive
- lev-qo1m.13 — Audit plugins/* legacy (20+ dirs)
- lev-qo1m.14 — ships-with flag implementation

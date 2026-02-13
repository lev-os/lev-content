# Core System Contract (LOCKED)

> Extracted from `docs/01-architecture.md`. Every module, plugin, and external package follows these rules. No exceptions.

---

## Architectural Standards

| Standard | Rule | Test |
|----------|------|------|
| **SRP** | One job per module | "Describe what this module does in ≤8 words without 'and'" |
| **Standalone Value** | Every package works independently via `npm install` | "Can an external dev use this without knowing about lev?" |
| **Zero Sibling Coupling** | Modules don't import from siblings. They declare capabilities, poly discovers. | "Does this module import from ../other-module?" → NO |
| **Config Declaration** | Modules self-declare via config.yaml with namespace sections | "Does this module have a config.yaml with its capabilities?" |
| **XDG Compliance** | No ~/.lev pollution. ~/.config/lev/, ~/.local/share/lev/, ~/.cache/lev/ | "Any state written outside XDG paths?" → NO |
| **LevEvent** | All inter-module events use LevEvent schema (lightweight, lev-specific — replaces legacy event schemas) | "Does this event have version/type/source/ts/time/id/data?" |
| **Vernacular** | Uses locked 5+7 term taxonomy everywhere (5 core + 7 reactive) | "Any file using 'hook' ambiguously?" → fix to Watcher/Trigger/Action |

---

## Config Resolution Chain (Fractal)

Lev discovers configuration from multiple sources, deep-merged in this order (last wins):

```
1. System:     ~/.config/lev/config.yaml          (XDG global defaults)
2. Project:    <project>/.lev/config.yaml          (project-level)
3. Module:     <module>/config.yaml                (self-declaration)
4. Environment: LEV_* env vars                     (runtime override)
```

Alternative discovery (external packages):
```
- package.json → "lev": { ... }       (npm packages)
- lev.yaml                            (standalone projects)
- .lev/config.yaml                    (project dir)
- <module>/config.yaml                (core modules)
- <module>/config/<namespace>.yaml    (large modules, split per namespace)
```

All resolve identically. Resolver checks single file first, then config/ directory.

---

## Config Namespace Declarations

Each module declares its capabilities via namespaced sections in config.yaml:

```yaml
# Example: core/index/config.yaml
poly:                          # Cross-language bindings
  sdk:
    find: { handler: "src/find.ts" }
  daemon:
    attach_to: python-daemon
    port: 9852

index:                         # Search backend registration
  backends: [leann, ck, mgrep]
  collections: [codebase, docs]

daemon:                        # Process supervision request
  autostart: true
  health: http://localhost:9852/health
```

**Namespaces (locked, triggers TBD):**

| Namespace | What It Declares | Consumer |
|-----------|-----------------|----------|
| `poly:` | Cross-language SDK bindings, codegen targets | polyglot-runners |
| `index:` | Search backends, collections, custom providers | core/index |
| `daemon:` | Service lifecycle intent (health, autostart, attach) | polyglot-runners process adapter |
| `memory:` | Storage backends, embedding config | core/memory |
| `skills:` | Capability registry, skill metadata | skills-registry |
| `triggers:` | Event sources, watchers, trigger rules, actions | observe/ (Bus) + triggers/ (rules) |

---

## Registration Flow

```
lev install <thing>
    │
    ▼
  Resolve config (package.json/lev.yaml/.lev/config.yaml)
    │
    ▼
  Parse namespace declarations
    │
    ├── poly: → polyglot-runners binds CLI/SDK/MCP/daemon surfaces
    ├── index: → adds to search registry
    ├── daemon: → contributes lifecycle intent for poly process management
    ├── memory: → registers storage backend
    ├── skills: → registers capabilities
    └── triggers: → registers event subscriptions (TBD)
    │
    ▼
  Module is in the ecosystem. Works from any directory.
```

---

## Poly Surface Contract (Required)

Any module or plugin exposing runnable surfaces MUST declare them in `config.yaml` under `poly:`.

- `poly.sdk` for root command bindings.
- `poly.cli` for standalone binaries.
- `poly.daemon` for long-running service lifecycle.
- `poly.mcp` for MCP tool surfaces.

`package.json` metadata (`leviathan`, `mcp`, `bin`) is descriptive and package-level only. Runtime binding authority is the merged poly registry (`core/polyglot-runners/.build/registry-runtime.yaml`).

Execution surface rule:
- `@lev-os/exec` is the canonical execution contract.
- CLI/MCP/daemon execution surfaces delegate to the same SDK path and remain thin transport wrappers.
- Wrapper layers must not duplicate orchestration/policy business logic that belongs in SDK core.

---

## Standalone Value Examples

| Module | Standalone JTBD | External Dev Usage |
|--------|----------------|-------------------|
| `daemon` | BD queue application service | Run and operate the BD task queue domain logic |
| `index` | Multi-backend search with RRF fusion | Search across codebases |
| `harness` | Hex execution engine with pluggable providers | Run prompts against any LLM |
| `polyglot-runners` | Binding + process lifecycle runtime | Generate typed bindings and supervise declared daemons |
| `config` | Fractal YAML config resolution | Config cascade for any project |
| `flowmind` | YAML workflow compiler | Compile declarative workflows |
| `coordination` | FlowMind-driven worker orchestration | Orchestrate agent workers via spawn/wait CLI steps |

---

## Plugins Layout (LOCKED — Session 5, 2026-01-31)

```
plugins/                                    # Extension layer
├── platforms/src/                          # IDE/gateway integrations (ships-with: true)
│   ├── config-generator.ts                 # shared base
│   ├── transcript-watcher.ts               # shared base
│   ├── bridge.ts                           # shared base
│   ├── cursor/                             # ships-with: true
│   ├── claude-code/                        # ships-with: true
│   └── openclaw/                           # ships-with: true
│       ├── config-generator.ts
│       ├── bridge.ts
│       ├── transcript-watcher.ts
│       └── config.yaml
│
├── sdlc/src/                               # SDLC EventProviders (ships-with: true)
│   └── eventproviders/
│       └── git.ts                          # GitEventProvider (from triggers/)
│
├── scheduling/src/                         # Time EventProviders (ships-with: true)
│   └── eventproviders/
│       └── cron.ts                         # CronEventProvider (from triggers/)
│
├── timetravel/src/                         # ships-with: true
├── workshop/src/                           # ships-with: true
├── automem/src/                            # ships-with: false (optional)
└── graphiti/src/                           # ships-with: false (optional)
```

**ships-with flag:** In each plugin's config.yaml:
```yaml
# plugins/sdlc/config.yaml
ships-with: true    # Included in default lev install
```

**Plugin surface types:**
- `config-generator` — emit config so lev works in platform environment
- `transcript-watcher` — scan conversation logs, emit LevEvents
- `bridge` — gateway ↔ harness middleware (platforms with gateways only)
- `eventproviders/` — translate native events → LevEvent

---

## Install Pipeline Layers

When a user runs `lev install <app>`, these layers activate:

```
lev install clawdbot
    │
    ▼
┌─ REGISTRY ─────────────────────────────────┐
│  polyglot-runners discovers available apps  │
└────────────────┬───────────────────────────┘
                 ▼
┌─ INSTALLER ────────────────────────────────┐
│  Resolves deps, downloads, validates       │
└────────────────┬───────────────────────────┘
                 ▼
┌─ CONFIG GENERATOR ─────────────────────────┐
│  plugins/platforms/clawdbot/ (target) emits │
│  gateway config, middleware, type bindings  │
└────────────────┬───────────────────────────┘
                 ▼
┌─ BRIDGE ───────────────────────────────────┐
│  Middleware connecting platform gateway     │
│  → harness execution (ws://→harness pipe)  │
└────────────────┬───────────────────────────┘
                 ▼
┌─ PROVIDER ─────────────────────────────────┐
│  harness/providers/claude-sdk.ts executes  │
│  prompts via Claude Agent SDK              │
└────────────────────────────────────────────┘
```

---

## Platform Bridge Pattern (OpenClaw)

Clawdbot (now OpenClaw) is the reference implementation of a Platform Bridge.

**What the bridge does:**
- Connects a platform's gateway (ws://127.0.0.1:18789) to harness execution
- Translates inbound messages → HarnessTask
- Translates HarnessResult → outbound platform messages
- Manages session state (JSONL transcripts)

**Interface:**
```typescript
interface PlatformBridge {
  readonly platform: string;          // 'openclaw' | 'native' | ...
  connect(gateway: GatewayConfig): Promise<void>;
  pipe(message: InboundMessage): HarnessTask;
  emit(result: HarnessResult): OutboundMessage;
  sessions: SessionManager;           // JSONL + sessions.json
}
```

**OpenClaw implementation flow:**
```
Gateway (ws://)
  ↓ InboundMessage (WhatsApp/Telegram/Discord)
Bridge (vendor/clawdbot-patch → plugins/platforms/openclaw/ target path)
  ↓ HarnessTask
Harness
  ↓ selects Provider
Provider (Claude Agent SDK)
  ↓ HarnessResult
Bridge
  ↓ OutboundMessage
Gateway (ws://) → channel
```

**Current state:** vendor/clawdbot-patch has this logic ad-hoc. Target: plugins/platforms/openclaw/bridge.ts

**To add a new platform:** Implement PlatformBridge interface. That's it.

---

## Refinement Passes

Architecture reconciliation happens in passes. Each pass has a specific lens:

| Pass | Lens | Focus | Status |
|------|------|-------|--------|
| **L1: Layout Coherence** | "Does the name match the job? Is it in the right place?" | Vernacular, renames, module locations, dead code removal | IN PROGRESS |
| **L2: Contract Compliance** | "Does every module follow the Core System Contract?" | SRP audit, standalone value check, config.yaml declarations, zero coupling | NOT STARTED |
| **L3: Integration Validation** | "Does everything work together end-to-end?" | Validation gates, TDD critical path, cross-module tests, smoke tests | NOT STARTED |

L1 must complete before L2. L2 must complete before L3.

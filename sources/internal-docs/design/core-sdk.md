# 05 - Core: SDK, Harness & Platforms

**Status**: ✅ LOCKED (Session 4-6) — execution in progress (hard-cut relocation complete for adapter sources)
**Epic**: lev-qo1m (Final Boss), lev-w7sm (Harness)
**Last Updated**: 2026-02-10

---

## Scope

**What this covers:**
- Execution harness (hex architecture)
- LLM providers (Claude SDK, OpenAI, AI SDK)
- Platform integrations (Cursor, Claude Code, OpenClaw)

**Modules:**
```
~/lev/core/
├── harness/            # Hex execution engine (renamed from agent-harness/)
│   └── src/
│       └── providers/  # LLM execution backends (renamed from adapters/)
├── agent-adapter/
│   └── src/
│       └── sync/       # Conversation sync runtime (platform adapters removed)
└── plugins/
    └── platforms/      # Canonical platform adapter home
```

---

## LOCKED VERNACULAR (5 Terms)

| Term | Meaning | Example |
|------|---------|---------|
| **Harness** | Hex execution engine (domain core) | Orchestrates prompt execution |
| **Provider** | LLM execution backend | ClaudeSDKProvider, ShellProvider |
| **Platform** | External system we integrate with | cursor/, claude-code/, openclaw/ |
| **Runner** | Build-time registry unit | Binary, daemon, SDK command |
| **Integration** | RETIRED — use Platform | — |

### What Changed
- ~~Adapter~~ → Provider (adapters/ was overloaded with hex pattern)
- ~~Client~~ → Platform (misleading term)
- ~~agent-harness~~ → harness (simpler)
- ~~integrations/~~ → plugins/platforms/ (canonical location)

---

## LOCKED DECISIONS

### Q1: agent-harness/ → harness/ ✅
**Decision**: Rename to `harness/` (task lev-qo1m.9)
**SRP**: "Hex execution engine" (3 words)

### Q2: adapters/ → providers/ ✅
**Decision**: Rename to `providers/` — LLM execution backends
**Contents**: claude-agent-sdk.ts, ai-sdk.ts, pi.ts, cli.ts, shell.ts

### Q3: agent-adapter/ fate ✅
**Decision**: target state is platform adapters under `plugins/platforms/`
- Event/hook functionality → events/ Bus
- Platform adapters currently in `core/agent-adapter/src/adapters/` → target `plugins/platforms/`
- Hard cut complete (2026-02-09): adapter implementations + `BaseAdapter` relocated to `plugins/platforms/` with no compatibility shim layer in `core/agent-adapter`.

### Q4: integrations/ → plugins/platforms/ ✅
**Decision**: platforms are plugins that ship-with: true
```yaml
# plugins/platforms/openclaw/config.yaml
ships-with: true
```

### Q5: Vernacular ✅
**Decision**: 5-term core taxonomy (see table above)

---

## Harness Architecture (Hex)

```
         ┌─────────────────────────┐
         │    Domain Core          │
         │  (Business Logic)       │
         └───────────┬─────────────┘
                     │
         ┌───────────▼─────────────┐
         │    Ports (Interfaces)   │
         │  - IHarness             │
         │  - IProvider            │
         └───────────┬─────────────┘
                     │
    ┌────────────────┼────────────────┐
    │                │                │
┌───▼────┐      ┌───▼────┐      ┌───▼────┐
│  CLI   │      │  SDK   │      │  API   │
│Provider│      │Provider│      │Provider│
└────────┘      └────────┘      └────────┘
```

---

## Platform Bridge Pattern

Platforms implement PlatformBridge interface:
```typescript
interface PlatformBridge {
  readonly platform: string;
  connect(gateway: GatewayConfig): Promise<void>;
  pipe(message: InboundMessage): HarnessTask;
  emit(result: HarnessResult): OutboundMessage;
  sessions: SessionManager;
}
```

**Surfaces per platform:**
- `config-generator` — emit config so lev works in platform environment
- `transcript-watcher` — scan conversation logs, emit LevEvents
- `bridge` — gateway ↔ harness middleware

---

## Epics

- lev-qo1m.9 — Rename agent-harness/ + adapters/ (APPROVED)
- lev-w7sm — Harness (absorbed lev-v2x9, lev-ccwu)
- lev-k8sb — OpenClaw Platform Integration (renamed from Clawdbot)

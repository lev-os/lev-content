# Clawdbot SDK Usage (Definitive)

**Created**: 2026-01-26
**Status**: CANONICAL
**Purpose**: Source of truth for clawdbot's relationship with lev SDK

---

## What Clawdbot Is

**Clawdbot is a FORK that uses lev SDK primitives, NOT a lev integration.**

```
~/clawd/vendor/clawdbot/       # Upstream fork (synced from github.com/clawdbot/clawdbot)
├── Uses: @anthropic-ai/claude-agent-sdk (direct dependency)
├── Uses: Lev SDK types (aspirational, via @canonical comment)
└── Does NOT use: Lev harness, lev exec, or any lev runtime primitives
```

**Key Distinction:**
- Clawdbot = standalone gateway (WhatsApp, Telegram, Discord, etc.)
- Lev = execution runtime (harness, adapters, flowmind)
- Integration = middleware in ~/clawd/tools/gateway-bridge/ (NOT in lev core)

---

## Session Management

### Architecture

Clawdbot has **complete session management** built-in. No lev dependency.

**Location**: `vendor/clawdbot/src/config/sessions/`

```typescript
// Session store (persistent)
~/.clawdbot/sessions.json                    // Main session registry
~/.clawdbot/agents/<agentId>/sessions/*.jsonl // Per-session JSONL transcripts

// Session management modules
src/config/sessions/store.ts                 // Load/save session registry
src/config/sessions/types.ts                 // SessionEntry, SessionScope
src/config/sessions/transcript.ts            // JSONL transcript I/O
src/config/sessions/metadata.ts              // Session metadata helpers
src/config/sessions/paths.ts                 // Path resolution
```

### SessionEntry Type

```typescript
// vendor/clawdbot/src/config/sessions/types.ts
export type SessionEntry = {
  sessionId: string;
  updatedAt: number;
  sessionFile?: string;                      // JSONL transcript path
  spawnedBy?: string;                        // Parent session (for sub-agents)
  chatType?: SessionChatType;
  model?: string;
  modelProvider?: string;

  // Delivery context (where to send replies)
  deliveryContext?: DeliveryContext;
  lastChannel?: SessionChannelId;
  lastTo?: string;
  lastAccountId?: string;
  lastThreadId?: string | number;

  // Usage tracking
  inputTokens?: number;
  outputTokens?: number;
  totalTokens?: number;
  contextTokens?: number;
  compactionCount?: number;

  // Session state
  queueMode?: 'steer' | 'followup' | 'collect' | 'queue' | 'interrupt';
  sendPolicy?: 'allow' | 'deny';

  // Skills snapshot (skills loaded in this session)
  skillsSnapshot?: SessionSkillSnapshot;
};
```

### Session Lifecycle

1. **Creation**: User sends message → gateway creates SessionEntry in `sessions.json`
2. **Persistence**: Conversation turns written to JSONL (`~/.clawdbot/agents/<agentId>/sessions/<sessionId>.jsonl`)
3. **Retrieval**: Next message loads SessionEntry + parses JSONL for history
4. **Delivery**: Response sent via `deliveryContext` (channel, to, accountId, threadId)
5. **Cleanup**: Inactive sessions archived (configurable TTL)

**Cache**: In-memory cache with 45s TTL (configurable via `CLAWDBOT_SESSION_CACHE_TTL_MS`)

---

## Claude Agent SDK Integration

### Direct SDK Usage (No Lev Harness)

**Location**: `vendor/clawdbot/src/agents/claude-sdk-runner/`

```typescript
// Main adapter
src/agents/claude-sdk-runner/adapter.ts      // runClaudeSdkAgent()
src/agents/claude-sdk-runner/session-bridge.ts // JSONL history parsing
src/agents/claude-sdk-runner/tool-wrapper.ts  // AgentTool → MCP bridge

// Dependency: @anthropic-ai/claude-agent-sdk
import { query } from '@anthropic-ai/claude-agent-sdk';
```

### How It's Wired

**Flow**:
```
User Message (WhatsApp)
  ↓
Gateway (src/gateway/server.impl.ts)
  ↓
Session Router (src/sessions/)
  ↓
Agent Runner (src/agents/claude-sdk-runner/adapter.ts)
  ↓
runClaudeSdkAgent()
  ├── Load session history (session-bridge.ts)
  ├── Build system prompt (src/agents/pi-embedded-runner/system-prompt.js)
  ├── Wrap tools (tool-wrapper.ts: AgentTool → MCP)
  └── Call SDK: query({ prompt, model, tools })
  ↓
Append to JSONL (session-bridge.ts: appendSessionTurns())
  ↓
Format response (src/gateway/)
  ↓
Deliver via channel (WhatsApp, Telegram, etc.)
```

### Key Functions

**1. runClaudeSdkAgent (adapter.ts)**
```typescript
export async function runClaudeSdkAgent(
  params: RunEmbeddedPiAgentParams,
): Promise<EmbeddedPiRunResult> {
  // 1. Load session history from JSONL
  const sessionHistoryPrompt = await buildSessionHistoryPrompt({
    sessionFile: params.sessionFile,
  });

  // 2. Wrap clawdbot tools as MCP
  const codingTools = createClawdbotCodingTools({ ... });
  const clawdbotMcp = createClawdbotMcpServerWithTools(codingTools);

  // 3. Call SDK query()
  for await (const message of query({
    prompt: userPrompt,
    model: sdkModel,
    options: {
      allowedTools: toolNames,
      mcpServers: [clawdbotMcp],
    },
  })) {
    // Stream results, emit progress
  }

  // 4. Append turns to JSONL
  await appendSessionTurns({
    sessionFile: params.sessionFile,
    userText: params.userText,
    assistantText: resultText,
  });

  return { success, output, ... };
}
```

**2. buildSessionHistoryPrompt (session-bridge.ts)**
```typescript
export async function buildSessionHistoryPrompt(
  opts: SessionHistoryOptions,
): Promise<string | null> {
  // Parse JSONL transcript
  const raw = await fs.readFile(opts.sessionFile, 'utf-8');
  const entries = raw.split(/\r?\n/)
    .filter(Boolean)
    .map(JSON.parse)
    .filter(entry => entry.type === 'message')
    .slice(-maxMessages);

  // Format as prompt
  return entries.map(e => `${e.role}: ${e.content}`).join('\n');
}
```

**3. createClawdbotMcpServerWithTools (tool-wrapper.ts)**
```typescript
export function createClawdbotMcpServerWithTools(
  tools: AnyAgentTool[],
  tracker: MessagingToolTracker,
): McpSdkServerConfigWithInstance {
  // Convert AgentTool (clawdbot) → SDK tool (MCP)
  const sdkTools = tools.map(tool => {
    const zodSchema = typeBoxToZod(tool.inputSchema);
    return tool({
      name: tool.name,
      description: tool.description,
      schema: z.object(zodSchema),
      execute: async (input) => {
        // Call original tool.execute()
        const result = await tool.execute(input);
        // Track messaging tools
        if (tool.name === 'sessions_send_message') {
          tracker.didSend = true;
        }
        return result;
      },
    });
  });

  return createSdkMcpServer({
    name: 'clawdbot',
    tools: sdkTools,
  });
}
```

---

## SDK Dependencies

### NPM Dependencies (package.json)

```json
{
  "dependencies": {
    "@anthropic-ai/claude-agent-sdk": "0.2.6",  // Direct SDK usage
    "@mariozechner/pi-agent-core": "0.49.2",    // Tool types (AgentTool)
    "@mariozechner/pi-coding-agent": "0.49.2",  // Legacy Pi agent types
    // ... (no lev dependencies)
  }
}
```

**No lev SDK dependency.** Clawdbot does NOT import from `@lev/*` packages.

### Aspirational Lev Types (Not Yet Used)

**Location**: `vendor/clawdbot/src/config/types.agents.ts`

```typescript
/**
 * Agent Configuration Types
 *
 * @canonical ~/lev/core/agent-adapter/src/clawdbot/types.ts
 *
 * Core types (AgentModelConfig, AgentBinding) are defined in lev's agent-adapter SDK.
 * This file extends those with clawdbot-specific fields (sandbox, heartbeat, etc.).
 *
 * Migration path:
 * 1. Core types → import from '../lev-types/types.js'
 * 2. Extended types → remain here with clawdbot deps
 */
```

**Status**: ASPIRATIONAL COMMENT ONLY

- No symlink exists at `src/lev-types/`
- No imports from lev SDK
- Types are duplicated locally in clawdbot

**Intended Design** (from comment):
```typescript
// Future (not implemented):
import { AgentConfig, AgentModelConfig } from '../lev-types/types.js';

// src/lev-types/ would be symlink to:
// ~/lev/core/agent-adapter/src/clawdbot/
```

---

## What Lev Must Provide (For Future Integration)

### Minimal SDK Surface

If clawdbot were to use lev SDK, it would need:

**1. Agent Configuration Types**
```typescript
// ~/lev/core/agent-adapter/src/clawdbot/types.ts
export interface AgentConfig {
  id: string;
  name?: string;
  workspace?: string;
  model?: AgentModelConfig;
  tools?: AgentToolsConfig;
}

export type AgentModelConfig = string | {
  primary?: string;
  fallbacks?: string[];
};
```

**2. Session Primitives (If Shared)**
```typescript
// ~/lev/core/agent-adapter/src/clawdbot/session.ts
export interface SessionDescriptor {
  id: string;
  file: string;  // JSONL path
  agent: string;
  channel: string;
  to: string;
}

export function parseSessionHistory(file: string): ConversationTurn[];
export function appendToSession(file: string, turn: ConversationTurn): Promise<void>;
```

**3. Event Primitives (If Integrated)**
```typescript
// ~/lev/core/agent-adapter/src/clawdbot/events.ts
export type AgentEvent =
  | { type: 'agent.started'; agentId: string }
  | { type: 'agent.completed'; agentId: string; result: unknown };

export function emitAgentEvent(event: AgentEvent): void;
export function onAgentEvent(handler: (event: AgentEvent) => void): void;
```

**NPM Package** (future):
```json
{
  "dependencies": {
    "@lev/agent-adapter": "^1.0.0"  // Shared types + primitives
  }
}
```

---

## The Mapping Work

### What Was Mapped

**Epic**: `clawd-ez6r` - SDK adapter integration
**Epic**: `clawd-4lgx` - Agent adapter event bridge

**Files Created**:
```
~/clawd/tools/harness/
├── sdk-adapter.ts                 # Claude Agent SDK → harness adapter
├── agent-adapter-events.ts        # Event bridge (SDK → EventBus)
├── adapter-parity.test.ts         # Adapter equivalence tests
└── harness.ts                     # Core harness interface

~/clawd/vendor/clawdbot/src/agents/claude-sdk-runner/
├── adapter.ts                     # SDK execution (clawdbot fork)
├── session-bridge.ts              # JSONL history parsing
└── tool-wrapper.ts                # AgentTool → MCP bridge
```

**What Was Mapped**:

1. **AgentTool (Pi) → MCP Tool (SDK)**
   - Clawdbot tools use TypeBox schemas
   - SDK expects Zod schemas
   - `tool-wrapper.ts` does conversion: `typeBoxToZod()`

2. **JSONL Session → SDK Prompt History**
   - Clawdbot stores turns as JSONL
   - SDK needs formatted prompt string
   - `session-bridge.ts` parses JSONL → "User: ...\nAssistant: ..."

3. **Harness Events → Agent Adapter Events**
   - Harness emits: `job.started`, `job.completed`, etc.
   - Agent adapter needs: `agent.started`, `agent.completed`, etc.
   - `agent-adapter-events.ts` subscribes to harness EventBus → emits adapter events

4. **Promise Detection (Ralph Loop)**
   - Task has `<promise>TOKEN</promise>` in prompt
   - SDK streams output → check for TOKEN in assistant text
   - Mark `promiseDetected: true` if found

5. **Multi-Iteration Retry**
   - If promise NOT detected → retry up to `maxIterations`
   - SDK returns partial output → harness retries with adversarial prompt
   - `sdk-adapter.ts` implements retry loop

---

## Current Reality vs Aspiration

### Current (2026-01-26)

| Component | Reality |
|-----------|---------|
| **Session Management** | Clawdbot owns (no lev dependency) |
| **Agent Config Types** | Duplicated (no shared SDK) |
| **SDK Integration** | Direct (clawdbot → @anthropic-ai/claude-agent-sdk) |
| **Tool Wrapping** | AgentTool → MCP (in clawdbot fork) |
| **Event Bridge** | Exists in tools/harness/ (NOT in clawdbot) |
| **Lev Types** | Aspirational @canonical comment (NOT implemented) |

### Aspiration (Future)

| Component | Goal |
|-----------|------|
| **Session Management** | Clawdbot owns (remains unchanged) |
| **Agent Config Types** | Import from @lev/agent-adapter |
| **SDK Integration** | Unchanged (clawdbot owns) |
| **Tool Wrapping** | Unchanged (clawdbot owns) |
| **Event Bridge** | Optional import from @lev/agent-adapter |
| **Lev Types** | Symlink: src/lev-types/ → ~/lev/core/agent-adapter/src/clawdbot/ |

**Key Decision**: Clawdbot can use lev SDK types, but does NOT depend on lev runtime (harness, exec, flowmind).

---

## Integration Points (Where Lev Connects)

### Middleware (NOT in lev core)

**Location**: `~/clawd/tools/gateway-bridge/` (to be created)

```
Clawdbot Gateway (vendor/clawdbot/)
  ↓ WebSocket
Middleware (tools/gateway-bridge/)
  ↓ IPC/API
Lev Harness (~/lev/core/agent-harness/)
  ↓ Provider
Claude Agent SDK (tools/harness/sdk-adapter.ts)
```

**Middleware responsibilities**:
- Listen to clawdbot gateway WS events
- Route incoming messages to lev exec
- Format lev results for gateway delivery
- Sync session state (if needed)

**NOT in lev core**: Middleware is clawdbot-specific (stays in clawd repo).

---

## Validation

### How to Verify This Doc

```bash
# 1. Check session management (clawdbot owns)
ls -la ~/.clawdbot/sessions.json
ls -la ~/.clawdbot/agents/*/sessions/*.jsonl

# 2. Check SDK integration (direct dependency)
grep "@anthropic-ai/claude-agent-sdk" vendor/clawdbot/package.json

# 3. Check for lev dependencies (should be NONE)
grep "@lev" vendor/clawdbot/package.json

# 4. Check symlink status (should NOT exist)
ls -la vendor/clawdbot/src/lev-types/

# 5. Check @canonical comment (aspirational)
grep -n "@canonical.*lev" vendor/clawdbot/src/config/types.agents.ts

# 6. Check harness adapter (exists in tools/harness/)
ls -la tools/harness/sdk-adapter.ts
ls -la tools/harness/agent-adapter-events.ts
```

### Expected Output

```
✅ ~/.clawdbot/sessions.json exists
✅ "@anthropic-ai/claude-agent-sdk": "0.2.6" found
❌ No @lev dependencies in package.json
❌ vendor/clawdbot/src/lev-types/ does not exist
✅ @canonical comment found in types.agents.ts
✅ tools/harness/sdk-adapter.ts exists
```

---

## Summary

**Clawdbot**:
- Standalone gateway fork
- Owns session management (JSONL + sessions.json)
- Direct SDK usage (@anthropic-ai/claude-agent-sdk)
- No lev runtime dependency
- Aspirational comment about lev types (NOT implemented)

**Lev SDK** (future):
- Can provide shared types (@lev/agent-adapter)
- Does NOT provide session management (clawdbot owns)
- Does NOT provide SDK integration (clawdbot owns)

**Integration** (middleware):
- Lives in ~/clawd/tools/gateway-bridge/
- NOT in lev core
- Bridges clawdbot WS ↔ lev harness

**Mapping Work**:
- AgentTool → MCP (tool-wrapper.ts)
- JSONL → Prompt history (session-bridge.ts)
- Harness events → Adapter events (agent-adapter-events.ts)
- Promise detection for ralph loop
- Multi-iteration retry logic

---

**Last Updated**: 2026-01-26
**Epic**: clawd-ez6r (SDK adapter), clawd-4lgx (event bridge)
**Status**: CANONICAL (source of truth for clawdbot build out)

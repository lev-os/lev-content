# Spec: IAbstractHooks Phase 1 Extension

**ID:** spec-hooks-phase1
**Status:** Draft
**Owner:** Lev Core
**Created:** 2026-02-03
**Updated:** 2026-02-03
**BD Epic:** lev-6zih (Task T1)

---

## 1. Business Case

### Problem Statement

The current `IAbstractHooks` interface has 15 events in 4 categories (session, tool, task, validation). Research across 10 IDE/CLI platforms identified 39 total events that enable comprehensive agent lifecycle instrumentation.

**Current coverage gaps:**
- No prompt lifecycle events (can't intercept before LLM query)
- No subagent/team events (can't track spawned agents)
- No context management events (can't hook compaction)

### User Value

Extending IAbstractHooks enables:
1. **Self-learning pipeline** - Hook session_end for triplet extraction
2. **Team mode observability** - Track subagent spawn/stop events
3. **Context management** - Hook pre_compact for memory injection
4. **Prompt validation** - Block/modify prompts before submission

### Strategic Alignment

- **Self-Learning Pipeline:** Phase 1 requires session_end hook for analysis
- **Research Foundation:** Hook systems research (2026-02-03) mapped 39 events
- **Platform Parity:** Cursor has all Phase 1 events; this achieves parity

### Alternatives Considered

| Alternative | Decision | Rationale |
|-------------|----------|-----------|
| Create new interface | Rejected | PATCH > ADD; extend existing |
| Add all 39 events at once | Rejected | Scope creep; Phase 1 focuses on essentials |
| Platform-specific hooks only | Rejected | Abstract layer enables multi-platform |

---

## 2. Architecture Overview

### System Context

**Current IAbstractHooks (15 events):**
```typescript
SessionEvent:     session_start | session_checkpoint | session_handoff | session_end
ToolEvent:        pre_tool_use | post_tool_use | tool_error
TaskEvent:        task_start | task_progress | task_complete | task_blocked
ValidationEvent:  dor_check | validation_pass | validation_fail | escalation
```

**Phase 1 Extension (+9 events = 24 total):**
```typescript
PromptEvent:      prompt_submit | prompt_rendered | prompt_complete     // NEW
SubagentEvent:    subagent_spawn | subagent_start | subagent_stop | subagent_message  // NEW
ContextEvent:     pre_compact | post_compact | context_overflow         // NEW
```

### Component Design

#### File: `core/agent-adapter/src/hooks/abstract-hooks.ts`

**Changes (PATCH):**

1. Add new event type definitions
2. Add new category to `LifecycleCategory`
3. Add payload interfaces for new events
4. Add event arrays for new categories
5. Update `ALL_LIFECYCLE_EVENTS`

#### File: `core/agent-adapter/src/hooks/claude-code-hooks.ts`

**Changes (PATCH):**

Add mappings for new events:

```typescript
const CLAUDE_CODE_HOOK_MAP: Record<string, string> = {
  // Existing mappings...

  // NEW: Prompt events
  prompt_submit: 'UserPromptSubmit',
  prompt_rendered: 'Notification',
  prompt_complete: 'Notification',

  // NEW: Subagent events
  subagent_spawn: 'Notification',
  subagent_start: 'Notification',
  subagent_stop: 'SubagentStop',
  subagent_message: 'Notification',

  // NEW: Context events
  pre_compact: 'PreCompact',
  post_compact: 'Notification',
  context_overflow: 'Notification'
};
```

#### File: `core/agent-adapter/src/hooks/index.ts`

**Changes (PATCH):**

Export new types:

```typescript
export {
  // Existing exports...

  // NEW: Phase 1 types
  PromptEvent,
  SubagentEvent,
  ContextEvent,
  PromptSubmitPayload,
  PromptRenderedPayload,
  SubagentSpawnPayload,
  PreCompactPayload,

  // NEW: Event arrays
  ALL_PROMPT_EVENTS,
  ALL_SUBAGENT_EVENTS,
  ALL_CONTEXT_EVENTS
} from './abstract-hooks.js';
```

### Technology Choices

| Choice | Rationale |
|--------|-----------|
| Extend existing types | PATCH > ADD principle |
| Union types for events | Consistent with existing pattern |
| Optional payloads | Backwards compatible |

### Interfaces

#### New Event Types

```typescript
export type PromptEvent =
  | 'prompt_submit'      // Before prompt sent to LLM, can block
  | 'prompt_rendered'    // After template rendering, can modify
  | 'prompt_complete';   // After LLM response received

export type SubagentEvent =
  | 'subagent_spawn'     // Before spawning subagent
  | 'subagent_start'     // Subagent begins execution
  | 'subagent_stop'      // Subagent completes, can continue
  | 'subagent_message';  // Inter-agent communication

export type ContextEvent =
  | 'pre_compact'        // Before context compaction
  | 'post_compact'       // After context compaction
  | 'context_overflow';  // Context window approaching limit
```

#### New Payload Types

```typescript
export interface PromptSubmitPayload extends HookPayload {
  prompt: string;
  context_tokens: number;
  max_tokens: number;
  can_block: boolean;
}

export interface PromptRenderedPayload extends HookPayload {
  original_prompt: string;
  rendered_prompt: string;
  template_name?: string;
  variables?: Record<string, unknown>;
}

export interface PromptCompletePayload extends HookPayload {
  prompt: string;
  response: string;
  tokens_used: number;
  duration_ms: number;
  model: string;
}

export interface SubagentSpawnPayload extends HookPayload {
  subagent_id: string;
  subagent_type: string;
  parent_session_id: string;
  prompt: string;
  allowed_tools?: string[];
}

export interface SubagentStartPayload extends HookPayload {
  subagent_id: string;
  subagent_type: string;
}

export interface SubagentStopPayload extends HookPayload {
  subagent_id: string;
  result: unknown;
  duration_ms: number;
  can_continue: boolean;
}

export interface SubagentMessagePayload extends HookPayload {
  from_id: string;
  to_id: string;
  message_type: 'request' | 'response' | 'broadcast';
  content: unknown;
}

export interface PreCompactPayload extends HookPayload {
  current_tokens: number;
  max_tokens: number;
  compaction_strategy: 'summary' | 'truncate' | 'selective';
  messages_to_compact: number;
}

export interface PostCompactPayload extends HookPayload {
  tokens_before: number;
  tokens_after: number;
  messages_removed: number;
  summary?: string;
}

export interface ContextOverflowPayload extends HookPayload {
  current_tokens: number;
  max_tokens: number;
  overflow_amount: number;
  suggested_action: 'compact' | 'handoff' | 'summarize';
}
```

---

## 3. Requirements

### Functional Requirements

**MUST:**
- [ ] Add PromptEvent types (prompt_submit, prompt_rendered, prompt_complete)
- [ ] Add SubagentEvent types (subagent_spawn, subagent_start, subagent_stop, subagent_message)
- [ ] Add ContextEvent types (pre_compact, post_compact, context_overflow)
- [ ] Add payload interfaces for all new events
- [ ] Update Claude Code hook mappings
- [ ] Export new types from index.ts
- [ ] Maintain backwards compatibility

**SHOULD:**
- [ ] Add blocking support for prompt_submit (can_block: boolean)
- [ ] Add continuation support for subagent_stop (can_continue: boolean)
- [ ] Document each new event with JSDoc

**MAY:**
- [ ] Add platform support matrix in comments
- [ ] Add example handlers for common use cases

### Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Type safety | TypeScript strict | Catch errors at compile time |
| Bundle size | +<2KB | Minimal overhead |
| Backwards compat | 100% | Existing code must work |

### Constraints

- **PATCH only** - No new files, extend existing
- **Backwards compatible** - Existing handlers continue working
- **Platform-specific mapping** - Each adapter maps to its hooks

### Dependencies

| Dependency | Status | Notes |
|------------|--------|-------|
| `abstract-hooks.ts` | Exists | Extend this file |
| `claude-code-hooks.ts` | Exists | Add mappings |
| `index.ts` | Exists | Export new types |

---

## 4. Validation Gates

### Acceptance Criteria

- [ ] All 9 new events defined in abstract-hooks.ts
- [ ] All 9 new events mapped in claude-code-hooks.ts
- [ ] All new types exported from index.ts
- [ ] TypeScript compiles without errors
- [ ] Existing tests pass (no regressions)
- [ ] New events can be emitted and handled

### Testing Strategy

| Test Type | Scope | Automation |
|-----------|-------|------------|
| Unit | Each new event type | Jest |
| Integration | Emit → Handle flow | Jest |
| Type | TypeScript compilation | tsc --noEmit |
| Regression | Existing 15 events | Existing tests |

### Success Metrics

- **Type coverage:** 100% of new events have payload types
- **Mapping coverage:** 100% of new events mapped in ClaudeCodeHooks
- **Backwards compatibility:** 0 breaking changes

### Rollout Plan

**Single PR:**
1. PATCH abstract-hooks.ts with new types
2. PATCH claude-code-hooks.ts with mappings
3. PATCH index.ts with exports
4. Run tests
5. Commit per spec

---

## 5. Implementation Tracking

### BD Task

**Epic:** lev-6zih
**Task:** T1 - IAbstractHooks Phase 1 Extension
**Priority:** P0

### Files to Modify

| File | Change Type | Lines Est |
|------|------------|-----------|
| `core/agent-adapter/src/hooks/abstract-hooks.ts` | PATCH | +80 |
| `core/agent-adapter/src/hooks/claude-code-hooks.ts` | PATCH | +15 |
| `core/agent-adapter/src/hooks/index.ts` | PATCH | +20 |

### Related Artifacts

- **Research:** `~/lev/docs/architecture/hook-systems-research-2026-02-03.md`
- **Parent Spec:** `~/lev/.lev/pm/specs/spec-self-learning-pipeline.md`
- **Abstract Hooks:** `~/lev/core/agent-adapter/src/hooks/abstract-hooks.ts`

---

## Appendix

### Platform Support Matrix

| Event | Claude Code | Cursor | SWE-agent | Aider |
|-------|------------|--------|-----------|-------|
| prompt_submit | UserPromptSubmit | beforeSubmitPrompt | on_model_query | - |
| prompt_rendered | - | - | - | - |
| prompt_complete | - | afterAgentResponse | on_step_done | - |
| subagent_spawn | - | subagentStart | - | - |
| subagent_start | - | - | - | - |
| subagent_stop | SubagentStop | subagentStop | - | - |
| subagent_message | - | - | - | - |
| pre_compact | PreCompact | preCompact | - | - |
| post_compact | - | - | - | - |
| context_overflow | - | - | - | - |

### Phase 2 Events (Future)

```typescript
// File operations (from Cursor)
FileEvent: pre_file_read | post_file_read | pre_file_write | post_file_write

// Shell operations (from Cursor)
ShellEvent: pre_shell_execute | post_shell_execute

// MCP operations (from Claude Code, Cursor)
McpEvent: pre_mcp_tool | post_mcp_tool

// Model operations (from SWE-agent)
ModelEvent: pre_model_query | post_model_query | model_thinking

// Lifecycle operations (from SWE-agent, PyTorch Lightning)
SetupEvent: setup_start | setup_complete | teardown_start | teardown_complete
```

### Implementation Snippet

```typescript
// abstract-hooks.ts additions

// New category
export type LifecycleCategory =
  | 'session'
  | 'tool'
  | 'task'
  | 'validation'
  | 'prompt'      // NEW
  | 'subagent'    // NEW
  | 'context';    // NEW

// New event types
export type PromptEvent = 'prompt_submit' | 'prompt_rendered' | 'prompt_complete';
export type SubagentEvent = 'subagent_spawn' | 'subagent_start' | 'subagent_stop' | 'subagent_message';
export type ContextEvent = 'pre_compact' | 'post_compact' | 'context_overflow';

// Updated union
export type LifecycleEvent =
  | SessionEvent
  | ToolEvent
  | TaskEvent
  | ValidationEvent
  | PromptEvent      // NEW
  | SubagentEvent    // NEW
  | ContextEvent;    // NEW

// New arrays
export const ALL_PROMPT_EVENTS: PromptEvent[] = [
  'prompt_submit', 'prompt_rendered', 'prompt_complete'
];

export const ALL_SUBAGENT_EVENTS: SubagentEvent[] = [
  'subagent_spawn', 'subagent_start', 'subagent_stop', 'subagent_message'
];

export const ALL_CONTEXT_EVENTS: ContextEvent[] = [
  'pre_compact', 'post_compact', 'context_overflow'
];

// Updated ALL_LIFECYCLE_EVENTS
export const ALL_LIFECYCLE_EVENTS: LifecycleEvent[] = [
  ...ALL_SESSION_EVENTS,
  ...ALL_TOOL_EVENTS,
  ...ALL_TASK_EVENTS,
  ...ALL_VALIDATION_EVENTS,
  ...ALL_PROMPT_EVENTS,      // NEW
  ...ALL_SUBAGENT_EVENTS,    // NEW
  ...ALL_CONTEXT_EVENTS      // NEW
];
```

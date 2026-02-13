# Spec: Self-Learning Pipeline

**ID:** spec-self-learning
**Status:** Draft
**Owner:** Lev Core
**Created:** 2026-02-03
**Updated:** 2026-02-03
**BD Epic:** lev-6zih

---

## 1. Business Case

### Problem Statement

Lev's learning pipeline has two missing pieces that prevent a closed feedback loop:

1. **No training data extraction** - Session transcripts exist but aren't converted to structured triplets (prompt, response, reward) for analysis
2. **No prompt optimization** - Even when patterns are detected, system prompts aren't automatically improved

**Current State (Open Loop):**
```
Session → events.jsonl → lev-learn detects patterns → reports
                                                          ↓
                                                   (manual action required)
```

**Desired State (Human-in-the-Loop Closed Loop):**
```
Session → events.jsonl → TraceAdapter → triplets → Analysis → PROPOSALS
     ↑                                                              ↓
     └──────────── human approves → improved prompts ←──────────────┘
```

### User Value

1. **System learns from sessions** - Successful patterns get reinforced, failures get addressed
2. **Human-in-the-loop control** - No autonomous changes; proposals require approval
3. **Measurable improvement** - Quantitative triplet data enables A/B testing
4. **Reduced manual work** - `/self-learn` skill surfaces actionable proposals

### Strategic Alignment

- **Self-Learning OS MVP:** Fills Phase 4.5 (Training Data) and Phase 4.6 (Prompt Optimization)
- **Agent Lightning Integration:** Extracts 4 patterns (not full framework adoption)
- **Memory Architecture:** Extends HybridMemoryOrchestrator (doesn't replace it)

### Alternatives Considered

| Alternative | Decision | Rationale |
|-------------|----------|-----------|
| Full agent-lightning adoption | Rejected | Python-heavy, requires VERL/PyTorch; Lev is TypeScript-first |
| Build unified store from scratch | Rejected | `HybridMemoryOrchestrator` already exists with 3 backends |
| Auto-apply APO | Rejected | User wants human-in-the-loop; proposals require approval |
| Defer to later phase | Rejected | Weight System (Phase 4) needs numeric signals |

---

## 2. Architecture Overview

### System Context

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Claude Code Session                               │
│  ┌─────────────┐   ┌──────────────┐   ┌─────────────┐   ┌───────────────┐  │
│  │UserPromptSub│   │  PreToolUse  │   │  PostToolUse│   │  SessionEnd   │  │
│  │   (memory)  │   │   (memory)   │   │  (rewards)  │   │  (review)     │  │
│  └──────┬──────┘   └──────┬───────┘   └──────┬──────┘   └───────┬───────┘  │
└─────────┼─────────────────┼──────────────────┼──────────────────┼──────────┘
          │                 │                  │                  │
          ▼                 ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    HybridMemoryOrchestrator (EXISTS)                        │
│  ┌───────────┐   ┌───────────┐   ┌───────────┐   ┌───────────────────────┐  │
│  │    BD     │   │  events   │   │transcripts│   │    memory backends    │  │
│  │ .beads/   │   │  .jsonl   │   │ ses_*.jsonl│   │ (Graphiti/OpenMem/AGI)│  │
│  └─────┬─────┘   └─────┬─────┘   └─────┬─────┘   └───────────┬───────────┘  │
└────────┼───────────────┼───────────────┼─────────────────────┼──────────────┘
         │               │               │                     │
         └───────────────┴───────┬───────┴─────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     TraceAdapter (NEW)                                       │
│    Transcript → Triplets (prompt, response, reward)                          │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Self-Learning Analyzer (NEW)                             │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────────────┐    │
│  │Pattern Detection│   │ Success/Failure │   │    Proposal Generator   │    │
│  │(existing)       │   │   Classifier    │   │  (replaces auto-APO)    │    │
│  └────────┬────────┘   └────────┬────────┘   └───────────┬─────────────┘    │
└───────────┼─────────────────────┼────────────────────────┼──────────────────┘
            │                     │                        │
            ▼                     ▼                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Proposals Directory                                 │
│    ~/.lev/proposals/self-learn/YYYY-MM-DD-{slug}.md                          │
│    ├── Surfaced via /self-learn skill                                        │
│    └── Human approves → Apply to CLAUDE.md / skills                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Component Design

#### 1. TraceAdapter

**Purpose:** Convert session transcripts to structured triplets

**Location:** `~/lev/core/memory/src/adapters/transcript-adapter.ts`

```typescript
interface Triplet {
  prompt: string;           // System prompt + user message
  response: string;         // Assistant's response
  reward: number;           // 0-1 outcome signal
  metadata: {
    session_id: string;
    bd_task?: string;
    tool_calls: string[];
    decision_point: 'tool_call' | 'code_edit' | 'task_complete';
  };
}

interface TranscriptAdapter {
  adapt(transcriptPath: string): Triplet[];
  extractReward(taskId: string): number;
  filterDecisionPoints(triplets: Triplet[]): Triplet[];
}
```

#### 2. RewardEmitter

**Purpose:** Emit numeric reward signals when BD tasks close

**Location:** `~/lev/core/memory/src/emitters/reward-emitter.ts`

```typescript
interface RewardDimension {
  name: string;     // "task_completion", "code_quality", "efficiency"
  value: number;    // 0.0 - 1.0
}

function emitReward(
  taskId: string,
  reward: number | Record<string, number>,
  primaryKey?: string
): void;

// BD close hook triggers:
// bd update task-123 --status closed → emitReward("task-123", 1.0)
// bd update task-123 --status closed --notes "partial" → emitReward("task-123", 0.7)
```

#### 3. SessionReviewHook

**Purpose:** Trigger analysis on session_end via IAbstractHooks

**Location:** Uses `onAfter('session_end')` from abstract-hooks.ts

```typescript
hooks.onAfter('session_end', async (event, payload) => {
  const triplets = await transcriptAdapter.adapt(payload.transcript_path);
  const analysis = await analyzer.analyze(triplets);

  if (analysis.hasProposal) {
    await writeProposal(analysis.proposal);
    await notify('New self-learning proposal ready for review');
  }
});
```

#### 4. ProposalGenerator (Replaces APO Auto-Apply)

**Purpose:** Generate human-reviewable proposals instead of auto-applying changes

**Location:** `~/lev/core/memory/src/algorithms/proposal-generator.ts`

```typescript
interface Proposal {
  id: string;
  created: Date;
  status: 'pending' | 'approved' | 'rejected';
  target: string;           // "~/.claude/CLAUDE.md" or "skills/X"
  change_type: 'add' | 'modify' | 'remove';
  evidence: Triplet[];      // Supporting data
  recommendation: string;   // What to change
  rationale: string;        // Why this helps
  rollback: string;         // How to undo
}

class ProposalGenerator {
  analyze(triplets: Triplet[]): Proposal | null;
  writeProposal(proposal: Proposal): void;
  listPending(): Proposal[];
  approve(id: string): void;
  reject(id: string, reason: string): void;
}
```

### Technology Choices

| Component | Technology | Rationale |
|-----------|------------|-----------|
| TraceAdapter | TypeScript | Lev is TypeScript-first |
| Triplet Storage | events.jsonl append | Consistent with existing telemetry |
| Proposals | Markdown files | Human-readable, git-trackable |
| Reward Signals | CloudEvents | Consistent with lev events |

### Interfaces

**IAbstractHooks Extension (Phase 1):**

```typescript
// New event categories
export type PromptEvent = 'prompt_submit' | 'prompt_rendered' | 'prompt_complete';
export type SubagentEvent = 'subagent_spawn' | 'subagent_start' | 'subagent_stop' | 'subagent_message';
export type ContextEvent = 'pre_compact' | 'post_compact' | 'context_overflow';

// New payload types
export interface PromptSubmitPayload extends HookPayload {
  prompt: string;
  context_size: number;
  can_block: boolean;
}

export interface SubagentSpawnPayload extends HookPayload {
  subagent_id: string;
  subagent_type: string;
  prompt: string;
}

export interface PreCompactPayload extends HookPayload {
  current_tokens: number;
  max_tokens: number;
  compaction_strategy: 'summary' | 'truncate' | 'selective';
}
```

---

## 3. Requirements

### Functional Requirements

**MUST:**
- [ ] Extract triplets from session transcripts (>80% decision point coverage)
- [ ] Emit reward signals when BD tasks close
- [ ] Generate proposals instead of auto-applying changes
- [ ] Surface proposals via `/self-learn` skill
- [ ] Integrate with session_end hook for automatic analysis
- [ ] Store proposals in `~/.lev/proposals/self-learn/`

**SHOULD:**
- [ ] Classify success/failure from session outcomes
- [ ] Correlate rewards with BD task status
- [ ] Track proposal approval/rejection rates
- [ ] Support rollback for approved changes

**MAY:**
- [ ] A/B test approved changes with confidence intervals
- [ ] Multi-dimensional rewards (task_completion, code_quality, efficiency)
- [ ] Integration with Langfuse for session observability (lev-czgj)

### Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Triplet extraction latency | <5s per session | Should not block session end |
| Proposal generation | <30s per batch | Async, not blocking |
| Reward correlation | Spearman >0.7 | Meaningful signal |
| Storage overhead | <10MB/week | Reasonable growth |

### Constraints

- **No autonomous changes:** All changes require human approval
- **TypeScript-first:** No Python dependencies in core pipeline
- **Extend, don't replace:** Use existing HybridMemoryOrchestrator
- **PATCH > ADD:** Extend IAbstractHooks, don't create parallel system

### Dependencies

| Dependency | Status | Notes |
|------------|--------|-------|
| `core/memory/src/orchestrator.ts` | Exists | HybridMemoryOrchestrator |
| `core/agent-adapter/src/hooks/abstract-hooks.ts` | Exists | 15 events, extend to 24 |
| BD lifecycle hooks | Exists | on_close for reward emission |
| `~/.claude/transcripts/` | Exists | Session transcript storage |

---

## 4. Validation Gates

### Acceptance Criteria

- [ ] Triplet extraction covers >80% of tool calls
- [ ] Reward signals correlate with BD task outcomes (Spearman >0.7)
- [ ] Proposal generation produces actionable recommendations
- [ ] `/self-learn` skill lists pending proposals
- [ ] Approved proposals correctly modify target files
- [ ] No autonomous changes without human approval

### Testing Strategy

| Test Type | Scope | Automation |
|-----------|-------|------------|
| Unit tests | TraceAdapter, RewardEmitter, ProposalGenerator | Jest |
| Integration | End-to-end triplet → proposal flow | Manual + snapshot |
| Validation | Human review of 10 proposals | Manual |
| Regression | Existing hooks continue working | CI |

### Success Metrics

**Quantitative:**
- Triplet coverage: >80%
- Reward correlation: Spearman >0.7
- Proposal acceptance rate: >50% (if <50%, proposals aren't useful)
- System improvement: Measurable reduction in repeated failures

**Qualitative:**
- Proposals are actionable and specific
- Human can understand proposal rationale
- Rollback instructions are accurate

### Rollout Plan

**Phase 1: Foundation (Week 1)**
- [ ] TraceAdapter implementation
- [ ] RewardEmitter + BD close hook
- [ ] IAbstractHooks Phase 1 extension

**Phase 2: Analysis (Week 2)**
- [ ] ProposalGenerator implementation
- [ ] `/self-learn` skill creation
- [ ] Proposal storage and listing

**Phase 3: Validation (Week 3)**
- [ ] End-to-end testing
- [ ] Human review of 10 proposals
- [ ] Documentation and handoff

---

## 5. Implementation Tracking

### BD Epic

**Epic ID:** lev-6zih
**Title:** Self-Learning: Agent Lightning Patterns
**Status:** Open (P0)

### Task Breakdown

| ID | Task | Depends | Priority |
|----|------|---------|----------|
| T1 | IAbstractHooks Phase 1 (Prompt, Subagent, Context) | - | P0 |
| T2 | TraceAdapter - triplet extraction | T1 | P0 |
| T3 | RewardEmitter + BD on_close hook | - | P0 |
| T4 | ProposalGenerator implementation | T2 | P0 |
| T5 | `/self-learn` skill | T4 | P1 |
| T6 | End-to-end testing | T1-T5 | P1 |

### Related Artifacts

- **Research:** `~/lev/workshop/intake/github/microsoft/agent-lightning/`
- **Proposal (original):** `~/lev/.lev/pm/proposals/20260201-agent-lightning-integration.md`
- **Hook research:** `~/lev/docs/architecture/hook-systems-research-2026-02-03.md`
- **Handoff:** `~/lev/.lev/pm/handoffs/20260203-050000-agent-lightning-self-learning.md`

### Status

**Current Phase:** Spec Draft
**Blockers:** None
**Next Steps:**
1. Review and approve spec
2. Create BD tasks from task breakdown
3. Start T1 (IAbstractHooks Phase 1)

---

## Appendix

### Research Summary

**Agent Lightning Patterns Extracted:**

| Pattern | Lev Implementation | Gap Filled |
|---------|-------------------|------------|
| TraceAdapter | `transcript-adapter.ts` | Training data extraction |
| Reward Emission | `reward-emitter.ts` | Numeric signals for weights |
| APO | `proposal-generator.ts` (human-in-the-loop) | Prompt optimization |
| Unified Store | HybridMemoryOrchestrator (exists) | None (already solved) |

### Decision Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2026-02-01 | Extract patterns, don't adopt framework | TypeScript-first, avoid VERL/PyTorch |
| 2026-02-01 | Unified Store already exists | HybridMemoryOrchestrator has 3 backends |
| 2026-02-03 | Human-in-the-loop for APO | User wants control over system changes |
| 2026-02-03 | PATCH > ADD for IAbstractHooks | Extend existing, don't create parallel |

### Open Questions

1. **Langfuse integration:** Defer to lev-czgj or include in this spec?
   - **Decision:** Defer, separate concern

2. **Reward signal granularity:** Single dimension or multi-dimensional?
   - **Decision:** Start with single (task_completion), expand if needed

3. **Proposal review UI:** CLI only or web dashboard?
   - **Decision:** CLI first (`/self-learn` skill), dashboard later

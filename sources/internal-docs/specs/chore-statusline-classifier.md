# Spec: Entity Lifecycle Status Line with Ollama Classification

**ID:** spec-lifecycle-statusline
**Status:** Draft
**Owner:** Lev Core / Hooks
**Created:** 2026-02-09
**Updated:** 2026-02-09
**Type:** Feature Spec

---

## 1. Business Case

### Problem Statement

The current Claude Code status line uses `bunx -y ccstatusline@latest` — a generic npm package with no awareness of the Leviathan entity lifecycle. Meanwhile, the `<lev>` footer protocol (`context/schema/examples/lev-footer-protocol.md`) defines 8 cognitive states that agents should track, but no infrastructure exists to classify user prompts into these states in real-time.

**Gaps:**
1. Status line displays generic info (model, cost) but no cognitive state
2. No automated lifecycle classification of user prompts
3. Footer protocol states exist in spec but aren't surfaced in tooling
4. Agents must self-report state — no independent classification for verification

### User Value

1. **Situational awareness** — Operator sees at a glance what cognitive mode the session is in
2. **Steering signal** — Downstream hooks/flows can consume lifecycle state for conditional behavior
3. **Session observability** — State history enables post-hoc analysis of cognitive flow patterns
4. **Protocol compliance** — Grounds footer protocol states in real classification, not self-report

### Strategic Alignment

- **Footer Protocol:** Implements the 8-state model from `lev-footer-protocol.md`
- **IAbstractHooks Phase 1:** Extends prompt lifecycle observability (spec-hooks-phase1)
- **Entity Lifecycle System:** Maps to cognitive state dimension of the entity lifecycle (spec-entity-lifecycle)
- **PATCH > ADD:** Extends existing hook infrastructure, doesn't create new systems

---

## 2. Architecture Overview

### Two-Tier Design

```
┌──────────────────────────────────────────────────────────────┐
│ Claude Code                                                  │
│                                                              │
│  User types prompt                                           │
│       │                                                      │
│       ▼                                                      │
│  ┌─────────────────────────────┐    ┌────────────────────┐   │
│  │ UserPromptSubmit Hook       │    │ Status Line Script  │   │
│  │ (lifecycle_classifier.py)   │    │ (statusline.py)     │   │
│  │                             │    │                     │   │
│  │ 1. Parse stdin JSON         │    │ 1. Read state file  │   │
│  │ 2. Classify via Ollama      │    │ 2. Read stdin JSON  │   │
│  │ 3. Write state file         │    │ 3. Format output    │   │
│  │ 4. Exit 0 (no stdout)      │    │ 4. Print one line   │   │
│  └──────────┬──────────────────┘    └────────┬───────────┘   │
│             │                                 │               │
│             ▼                                 ▼               │
│  ~/.cache/lev/lifecycle-state.json     stdout (status bar)   │
│                                                              │
└──────────────────────────────────────────────────────────────┘

External:
  Ollama (localhost:11434) ← qwen3:30b-a3b (fast MoE, ~150-300ms)
```

### Component Design

| Component | Responsibility | I/O |
|-----------|---------------|-----|
| `utils/ollama_classify.py` | HTTP client + classification logic | prompt → state |
| `UserPromptSubmit_lifecycle_classifier.py` | Hook: orchestrate classification on prompt submit | stdin JSON → state file |
| `statusline_lifecycle.py` | Display: read state + format | state file + stdin JSON → stdout |
| `test_lifecycle_e2e.py` | Validation: end-to-end test harness | synthetic data → results |

### Technology Choices

| Choice | Rationale |
|--------|-----------|
| Ollama (local) | No network latency, privacy-preserving, already deployed |
| qwen3:30b-a3b | Fast MoE model, ~150-300ms classification, low resource usage |
| Python (no deps) | Matches existing hook patterns, `uv run` compatible, stdlib only |
| File-based state | Decouples hook from status line (<5ms read), survives process restart |
| Keyword fallback | Graceful degradation when Ollama unavailable |

---

## 3. User Stories

### P1: Real-Time Lifecycle Display ⭐ MVP

**User Story:** As a developer using Claude Code, I want to see the current cognitive lifecycle state in the status bar so that I have situational awareness of the session mode.

**Acceptance Criteria:**
1. WHEN user submits a prompt THEN system SHALL classify it into one of 8 lifecycle states within 2 seconds
2. WHEN classification completes THEN system SHALL write state to `~/.cache/lev/lifecycle-state.json`
3. WHEN status line renders THEN system SHALL display lifecycle state with model name and context %
4. WHEN Ollama is unavailable THEN system SHALL fall back to keyword heuristic classification

**Independent Test:** Pipe synthetic prompt JSON to classifier hook, verify state file written; pipe mock stdin to status line, verify formatted output.

### P1: Graceful Degradation ⭐ MVP

**User Story:** As a developer, I want the status line to work even if Ollama is down so that my Claude Code experience is never degraded.

**Acceptance Criteria:**
1. WHEN Ollama is unreachable THEN system SHALL use keyword-based heuristic (no error, no hang)
2. WHEN hook times out THEN system SHALL exit 0 cleanly (no stdout, no block)
3. WHEN state file is stale (>5min) or missing THEN status line SHALL display "Idle" state

### P2: Feature Flag Control

**User Story:** As an operator, I want to disable lifecycle classification via environment variables without editing config files.

**Acceptance Criteria:**
1. WHEN `LEV_CM_HOOKS_ENABLED=false` THEN system SHALL skip all classification
2. WHEN `LEV_CM_HOOK_LIFECYCLE_CLASSIFIER_ENABLED=false` THEN system SHALL skip this hook only
3. WHEN `LEV_OLLAMA_MODEL` is set THEN system SHALL use the specified model

---

## 4. Detailed Design

### 4.1 State File Schema (`~/.cache/lev/lifecycle-state.json`)

```json
{
  "ts": "2026-02-09T20:15:30Z",
  "session_id": "abc123",
  "state": "execution",
  "confidence": 0.85,
  "model": "qwen3:30b-a3b",
  "elapsed_ms": 187,
  "prompt_preview": "implement the auth flow..."
}
```

### 4.2 Lifecycle States (from Footer Protocol)

| State | Display | Description |
|-------|---------|-------------|
| `brainstorm` | `🧠 Brainstorm` | Exploring possibilities, divergent thinking |
| `refinement` | `✏️ Refining` | Improving existing work, polish, iterate |
| `crystallization` | `💎 Crystallizing` | Solidifying decisions, finalize, document |
| `capture` | `📝 Capturing` | Recording information, save, persist |
| `context-healing` | `🔧 Healing` | Recovering lost context, load, resume |
| `planning` | `📋 Planning` | Creating action plans, decompose, sequence |
| `research` | `🔍 Researching` | Gathering information, search, analyze |
| `execution` | `⚡ Executing` | Performing actions, run, build, deploy |
| _(no state)_ | `⏳ Idle` | No recent classification |

### 4.3 Status Line Output Format

Single line: `{emoji} {State} | {Model} | {ctx_pct}% ctx`

Examples:
- `⚡ Executing | Opus | 25% ctx`
- `🔍 Researching | Opus | 12% ctx`
- `⏳ Idle | Opus | 3% ctx`

### 4.4 Ollama Classification Prompt

Constrained few-shot, single-token output:

```
Classify this user prompt into exactly one cognitive state.
States: brainstorm, refinement, crystallization, capture, context-healing, planning, research, execution

Examples:
"What if we split the config into layers?" → brainstorm
"Polish the error messages" → refinement
"Finalize the ADR for event bus" → crystallization
"Save this session as a handoff" → capture
"Where were we? Load last session" → context-healing
"Plan the migration from core/ to plugins/" → planning
"Search for how other frameworks handle this" → research
"Build the auth hook and test it" → execution

Prompt: "{user_prompt}"
State:
```

Settings: `temperature: 0`, `num_predict: 10`, `stream: false`, `keep_alive: -1`

### 4.5 Keyword Fallback Heuristic

When Ollama is unavailable, classify by keyword matching:

```python
KEYWORD_MAP = {
    "brainstorm": ["what if", "imagine", "could we", "ideas", "explore", "possibilities"],
    "refinement": ["polish", "improve", "refine", "tweak", "adjust", "clean up", "fix the"],
    "crystallization": ["finalize", "commit", "decide", "lock", "adr", "approve", "spec"],
    "capture": ["save", "record", "capture", "handoff", "checkpoint", "persist", "document this"],
    "context-healing": ["where were we", "resume", "load", "last session", "context", "remind me"],
    "planning": ["plan", "break down", "decompose", "sequence", "prioritize", "roadmap", "steps"],
    "research": ["search", "find", "look up", "how do", "what is", "analyze", "compare"],
    "execution": ["build", "implement", "create", "run", "deploy", "test", "write", "code"],
}
```

Score: count matching keywords per state, highest wins. Confidence = `match_count / total_keywords_in_state`. Default to `execution` if no matches.

### 4.6 Feature Flags

| Variable | Default | Description |
|----------|---------|-------------|
| `LEV_CM_HOOKS_ENABLED` | `true` | Global kill switch for all hooks |
| `LEV_CM_HOOK_LIFECYCLE_CLASSIFIER_ENABLED` | `true` | Per-hook toggle |
| `LEV_CM_HOOK_LIFECYCLE_TIMEOUT_MS` | `2000` | Per-hook timeout (ms) |
| `LEV_OLLAMA_ENDPOINT` | `http://localhost:11434` | Ollama API endpoint |
| `LEV_OLLAMA_MODEL` | `qwen3:30b-a3b` | Classification model |

---

## 5. File Manifest

### Files to Create

| # | File | Purpose | Lines Est |
|---|------|---------|-----------|
| 1 | `~/.claude/hooks/utils/ollama_classify.py` | Shared Ollama HTTP client + classifier | ~120 |
| 2 | `~/.claude/hooks/UserPromptSubmit_lifecycle_classifier.py` | Hook: classify prompt → write state file | ~100 |
| 3 | `~/.claude/hooks/statusline_lifecycle.py` | Status line: read state → format output | ~80 |
| 4 | `~/.claude/hooks/test_lifecycle_e2e.py` | E2E test harness | ~200 |

### Files to Modify

| File | Change |
|------|--------|
| `~/.claude/settings.json` | Add hook to `UserPromptSubmit` array, replace `statusLine.command` |

### Reuse Points

| Pattern | Source | Reuse |
|---------|--------|-------|
| `env_bool()` | `UserPromptSubmit_thinking_probe.py:32-37` | Copy verbatim |
| `timeout_ms()` | `UserPromptSubmit_thinking_probe.py:45-55` | Adapt for per-hook env var |
| `log_hook()` | `utils/hook_logger.py` | Import directly |
| Hook stdin parse | `UserPromptSubmit_thinking_probe.py:131-133` | Same pattern |
| Feature flag check | `UserPromptSubmit_thinking_probe.py:40-42` | Same 2-tier pattern |

---

## 6. Task Breakdown

### T1: Create `utils/ollama_classify.py`

**What:** Shared Ollama HTTP client with classification prompt and keyword fallback
**Where:** `~/.claude/hooks/utils/ollama_classify.py`
**Depends:** None
**Reuses:** `env_bool()`, `timeout_ms()` patterns from thinking_probe
**Done when:**
- [ ] `classify(prompt: str) -> dict` returns `{state, confidence, model, elapsed_ms}`
- [ ] Ollama HTTP call with configurable endpoint/model/timeout
- [ ] Keyword fallback when Ollama unreachable
- [ ] No external dependencies (stdlib `urllib` only)

### T2: Create `UserPromptSubmit_lifecycle_classifier.py`

**What:** Hook that classifies user prompt and writes state file
**Where:** `~/.claude/hooks/UserPromptSubmit_lifecycle_classifier.py`
**Depends:** T1
**Reuses:** Hook boilerplate from `UserPromptSubmit_thinking_probe.py`
**Done when:**
- [ ] Reads stdin JSON payload
- [ ] Checks feature flags (global + per-hook)
- [ ] Calls `ollama_classify.classify(prompt)`
- [ ] Writes `~/.cache/lev/lifecycle-state.json`
- [ ] Logs via `log_hook()`
- [ ] Exits 0 with no stdout (side-effect only hook)
- [ ] Skips prompts <10 chars

### T3: Create `statusline_lifecycle.py`

**What:** Status line script that reads state file and formats output
**Where:** `~/.claude/hooks/statusline_lifecycle.py`
**Depends:** T1 (state file schema), T2 (writes the state file)
**Reuses:** None (standalone reader)
**Done when:**
- [ ] Reads `~/.cache/lev/lifecycle-state.json` (graceful if missing/stale)
- [ ] Reads stdin JSON from Claude Code (model, context_window)
- [ ] Outputs single formatted line: `{emoji} {State} | {Model} | {pct}% ctx`
- [ ] Handles stale state (>5min) as "Idle"
- [ ] Completes in <5ms (file read only, no network)
- [ ] Python3 stdlib only (no `jq` dependency)

### T4: Create `test_lifecycle_e2e.py`

**What:** End-to-end test harness covering classification, status line, and roundtrip
**Where:** `~/.claude/hooks/test_lifecycle_e2e.py`
**Depends:** T1, T2, T3
**Reuses:** Transcript data from `~/.cache/lev/transcripts/`
**Done when:**
- [ ] 8 synthetic prompt test cases (1 per lifecycle state)
- [ ] Transcript extraction test (parse JSONL, classify user messages)
- [ ] Status line format verification
- [ ] Roundtrip test (mock state file → status line → verify output)
- [ ] Prints results table to stdout
- [ ] Saves results to `~/.cache/lev/lifecycle-test-results.json`
- [ ] Runnable via `uv run ~/.claude/hooks/test_lifecycle_e2e.py`

### T5: Install (settings.json update)

**What:** Wire hook and status line into Claude Code settings
**Where:** `~/.claude/settings.json`
**Depends:** T4 (tests pass first)
**Done when:**
- [ ] `UserPromptSubmit` array includes lifecycle classifier hook
- [ ] `statusLine.command` points to `statusline_lifecycle.py`
- [ ] Existing hooks/settings preserved (PATCH, not replace)

---

## 7. Validation Gates

### Acceptance Criteria (DoR)

```yaml
ENTRY:
  - Ollama running locally with qwen3:30b-a3b loaded
  - Existing hooks infrastructure intact (~/.claude/hooks/utils/)
  - Python 3.11+ available

DONE_STATE:
  - 4 new files created in ~/.claude/hooks/
  - settings.json updated with new hook + status line
  - E2E tests pass

CRITERIA:
  - Classification returns valid state for all 8 synthetic prompts
  - Keyword fallback works when Ollama is stopped
  - Status line renders in <5ms
  - Hook completes in <2s (Ollama) or <50ms (fallback)
  - No stdout from hook (side-effect only)
  - Feature flags disable/enable correctly

TOUCHPOINTS:
  - ~/.claude/settings.json (UserPromptSubmit, statusLine)
  - ~/.cache/lev/lifecycle-state.json (new state file)
  - ~/.claude/logs/hooks.log (logging via hook_logger)
  - Ollama API (localhost:11434)

INTEGRATION:
  - Follows existing hook patterns (thinking_probe, learn_router)
  - Uses shared utils (hook_logger, env_bool pattern)
  - State file consumable by future hooks/flows

E2E:
  - uv run ~/.claude/hooks/test_lifecycle_e2e.py
```

### Testing Strategy

| Test Type | Scope | Method |
|-----------|-------|--------|
| Unit | Keyword fallback classification | Synthetic prompts, no Ollama |
| Unit | State file read/write | Mock JSON |
| Integration | Ollama classification | Live Ollama call |
| Integration | Status line formatting | Pipe mock stdin |
| E2E | Full roundtrip | Hook → state file → status line |
| Regression | Existing hooks | Verify settings.json doesn't break existing hooks |

### Success Metrics

| Metric | Target |
|--------|--------|
| Classification accuracy (synthetic) | 8/8 states correct |
| Hook latency (Ollama up) | <2000ms p95 |
| Hook latency (Ollama down) | <50ms p95 |
| Status line latency | <5ms |
| Fallback activation | 0 crashes when Ollama unavailable |

---

## 8. Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Ollama not running | Medium | Keyword fallback, no crash |
| Model slow/unavailable | Medium | Configurable timeout, fallback |
| State file corruption | Low | Atomic write (write tmp + rename) |
| Hook blocks Claude Code | High | Strict timeout, exit 0 always |
| Wrong classification | Low | Confidence field enables downstream filtering |
| Status line flicker | Low | Stale threshold (5min), Idle default |

---

## 9. Execution Order

1. **T1** → `utils/ollama_classify.py` (dependency for everything)
2. **T2** → `UserPromptSubmit_lifecycle_classifier.py` (the hook)
3. **T3** → `statusline_lifecycle.py` (the display)
4. **T4** → `test_lifecycle_e2e.py` (the test)
5. **Run E2E** → `uv run ~/.claude/hooks/test_lifecycle_e2e.py`
6. **T5** → Update `~/.claude/settings.json` (install, only after tests pass)
7. **Verify** → Manual validation in live Claude Code session

---

## 10. Verification Checklist

```bash
# 1. Run E2E test suite
uv run ~/.claude/hooks/test_lifecycle_e2e.py

# 2. Manual classifier test
echo '{"prompt":"build the feature","session_id":"test"}' | \
  uv run ~/.claude/hooks/UserPromptSubmit_lifecycle_classifier.py

# 3. Check state file
cat ~/.cache/lev/lifecycle-state.json

# 4. Manual status line test
echo '{"model":{"display_name":"Opus"},"context_window":{"used_percentage":25}}' | \
  python3 ~/.claude/hooks/statusline_lifecycle.py

# 5. After install: submit prompt in Claude Code, verify status bar updates
```

---

## 11. Related Artifacts

| Artifact | Path |
|----------|------|
| Footer Protocol | `context/schema/examples/lev-footer-protocol.md` |
| Entity Lifecycle Spec | `.lev/pm/specs/spec-entity-lifecycle-system.md` |
| IAbstractHooks Phase 1 | `.lev/pm/specs/spec-iabstract-hooks-phase1.md` |
| Thinking Probe (pattern source) | `~/.claude/hooks/UserPromptSubmit_thinking_probe.py` |
| Hook Logger (shared util) | `~/.claude/hooks/utils/hook_logger.py` |
| Settings (target) | `~/.claude/settings.json` |

---

**Ready for:** Review and approval
**Next:** Implement T1-T5 sequentially, run E2E before install

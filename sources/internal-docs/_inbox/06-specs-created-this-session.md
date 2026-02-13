# Specs Created This Session

## Location: `~/.lev/pm/specs/`

### 1. spec-self-learning-pipeline.md
- Human-in-the-loop training data extraction
- TraceAdapter, RewardEmitter, ProposalGenerator
- Uses existing HybridMemoryOrchestrator

### 2. spec-iabstract-hooks-phase1.md
- 9 new events (15 → 24 total)
- PromptEvent: prompt_submit, prompt_rendered, prompt_complete
- SubagentEvent: subagent_spawn, subagent_start, subagent_stop, subagent_message
- ContextEvent: pre_compact, post_compact, context_overflow

### 3. spec-entity-lifecycle-system.md
- Meta-spec for specs
- Ephemeral as STATE not entity
- Command hierarchy (/work as main)
- When spec is required (decision tree)
- Event naming audit

## Status

All draft. Need review and approval before implementation.

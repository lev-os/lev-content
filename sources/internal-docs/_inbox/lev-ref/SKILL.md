---
name: lev-ref
description: Agentic UX design language and pattern library. Establishes CLI-as-Prompt paradigm where every CLI output is a prompt to the next agent. Includes Enriched Response Protocol, vernacular (Self-Injection, Teachable Touch, Bi-Directional), and implementation patterns for validation gates and trust calibration.
---

# The CLI-as-Prompt Paradigm

## Core Insight

**Every CLI output is a prompt to the next agent in the chain.**

When you return output from a CLI, you are talking to an agent. This inverts traditional CLI design: outputs are not just data for humans—they are inputs for the next reasoning step.

This unlocks **lazy evaluation through self-injection**: an agent can prompt itself with enriched context, deferring decisions until more information is available.

## Vernacular

| Term | Definition |
|------|------------|
| **CLI-as-Prompt** | Every CLI output is a prompt to the next agent. Design outputs as you would design prompts. |
| **Enriched Response** | Output + meta (follow-ups, docs, troubleshooting, confidence). The prompt is richer than raw data. |
| **Self-Injection** | Agent prompts itself with enriched context. Enables lazy evaluation—defer decisions until needed. |
| **Teachable Touch** | Every interaction teaches better usage. Errors become learning, corrections become preferences. |
| **Bi-Directional** | Both agent learns AND user is guided. Communication flows both ways simultaneously. |

## The Enriched Response Protocol

CLI output follows a simple rule: **inline conventions by default, JSON only when requested.**

### Detection Logic

```
if --json flag:  → JSON envelope (for traditional program integration)
else:            → Inline conventions (agent-native, human-readable)
```

**Why inline by default?** LLM agents read language, not data structures. The inline conventions ARE machine-readable—by machines that read. JSON exists for traditional programs that need structured parsing.

### JSON Envelope (traditional program integration)

```json
{
  "output": "Fixed 3 type errors in src/auth.ts",
  "meta": {
    "follow_ups": [
      "Run tests: pytest src/auth/",
      "Check related: src/middleware.ts"
    ],
    "troubleshooting": null,
    "docs": ["https://docs.example.com/auth"],
    "confidence": 0.92,
    "teach": "Type errors often cluster. Check imports when you see 3+ in one file.",
    "resume_context": {
      "session_id": "abc123",
      "checkpoint": "post-type-fix",
      "state_hash": "sha256:..."
    }
  }
}
```

### Inline Conventions (agent-native, default)

```
Fixed 3 type errors in src/auth.ts

[92% confident] All errors resolved (clear pattern match)

💡 Tip: Type errors often cluster. Check imports when you see 3+ in one file.

→ Next: Run tests with `pytest src/auth/`
→ Related: Check src/middleware.ts for similar issues

[Resume: agentctl resume abc123]
```

### Field Definitions (JSON schema / conceptual model)

These fields exist in both formats—JSON uses explicit keys, inline uses conventions:

| Field | JSON Key | Inline Convention | Required |
|-------|----------|-------------------|----------|
| Result | `output` | First line(s) of output | **Yes** |
| Next actions | `follow_ups` | `→ Next:` / `→ Related:` | No |
| Error help | `troubleshooting` | After `❌ Error:` | No |
| Documentation | `docs` | Inline links or `📚 Docs:` | No |
| Confidence | `confidence` | `[N% confident]` | No |
| Lesson | `teach` | `💡 Tip:` | No |
| Resume state | `resume_context` | `[Resume: X]` | No |

### Inline Convention Reference

For agent-native output, use these conventions:

| Convention | Meaning | Example |
|------------|---------|---------|
| `[N% confident]` | Confidence score | `[92% confident] Fix applied` |
| `💡 Tip:` | Teaching moment | `💡 Tip: Use --strict for more checks` |
| `→ Next:` | Follow-up action | `→ Next: Run tests` |
| `→ Related:` | Related context | `→ Related: Check auth.ts` |
| `[Resume: X]` | Resume command | `[Resume: agentctl resume abc123]` |
| `⚠️ Warning:` | Caution | `⚠️ Warning: This deletes data` |
| `❌ Error:` | Failure | `❌ Error: File not found` |
| `✓` / `✗` | Pass/fail | `✓ Tests passed (12/12)` |

## Design Principles

1. **Prompt-First Thinking**: Before outputting, ask "How would I prompt an agent with this information?"

2. **Dual Audience**: Every output serves both humans (readable) and agents (parseable). Never sacrifice one completely.

3. **Progressive Enrichment**: Start with output, add meta as complexity warrants. Simple commands need less meta.

4. **Teachable by Default**: Errors always include lessons. Success includes tips when non-obvious.

5. **Resumable State**: Long-running or interruptible operations include resume context.

---

# lev-ref - Agentic UX Pattern Library

## Purpose

lev-ref is a living database of agentic UX techniques derived from production CLI agents (Claude Code, Aider, Cursor, GitHub Copilot CLI) and academic research. It codifies patterns for bi-directional human-agent communication, transforming every CLI interaction into a teachable dialogue moment.

The patterns below are **implementations of the CLI-as-Prompt paradigm**. Each pattern shows how to structure output so it serves as an effective prompt to the next agent (human or machine) in the chain.

## When to Use This Skill

Apply lev-ref when:

- Designing CLI agent communication patterns
- Implementing feedback loops for agent learning
- Building validation gates for agentic workflows
- Creating teachable error messages
- Calibrating trust and confidence communication
- Researching agentic UX best practices

Semantic triggers:
- "How should the agent communicate progress?"
- "What makes a good error message for agents?"
- "How do we implement validation gates?"
- "Design the CLI feedback pattern"
- "Make this response teachable"

---

## Pattern Database

### 1. Bi-Directional Communication Patterns

#### Pattern: Annotated Progress
**Intent**: Show not just WHAT the agent is doing, but WHY and what evidence of success looks like.

```
[BAD]  Running task 3/12...
[GOOD] Analyzing auth module... Found 3 permission checks, verifying middleware chain
```

**Implementation**:
- Connect actions to observable outcomes
- Show reasoning alongside actions
- Create narrative arc users can follow

#### Pattern: Plan-Then-Act
**Intent**: Present reasoning and await approval before executing potentially irreversible actions.

```
I'm planning to:
1. Refactor auth middleware → consolidate 3 files into 1
2. Update import paths → 12 files affected
3. Add integration test → verify no regressions

Proceed? [Y/n/explain more]
```

**Implementation**:
- Separate planning phase from execution phase
- Create explicit approval checkpoint
- Allow modification before commitment

#### Pattern: Confidence Scoring
**Intent**: Attach explicit confidence levels to recommendations so users know when to verify.

```
[95% confident] Memory leak in loop at line 43 (clear pattern match)
[60% confident] Specific fix approach (multiple valid options exist)
```

**Implementation**:
- Score confidence per recommendation, not globally
- Explain factors affecting confidence
- Different thresholds trigger different approval requirements

---

### 2. Teachable Moment Patterns

#### Pattern: Layered Error Explanation
**Intent**: Transform errors from dead ends into learning opportunities.

**Structure**:
1. WHAT went wrong (the error)
2. WHY it went wrong (root cause)
3. HOW to avoid it (the lesson)

```
[Error] Branch 'develop-staging' not found

[Root Cause] You may have typed the branch name incorrectly, or it was deleted.

[Available Branches] develop, main, release/1.0

[Lesson] Use tab completion or `git branch -a` to see available branches before switching.
```

#### Pattern: Prompt Scaffolding
**Intent**: When user input is underspecified, ask structured questions that teach better prompting.

```
Your request "fix the auth system" could mean several things:
- Fix security vulnerabilities? (security focus)
- Refactor for clarity? (code quality focus)
- Add new auth methods? (feature focus)
- Improve performance? (optimization focus)

Which aspect concerns you most? This helps me understand your intent better.
```

**Implementation**:
- Present options that reveal possible interpretations
- Each option teaches a dimension of specificity
- Over time, users internalize these dimensions

#### Pattern: Correction as Teaching
**Intent**: When users correct agent output, treat it as a learning signal and acknowledge what was learned.

```
User: No, use 4-space indentation, not 2-space.

Agent: Got it. I'll use 4-space indentation for this file. I noticed the project
uses mixed indentation - should I standardize to 4-space everywhere, or is this
file-specific?

[Learning captured: User prefers 4-space indentation]
```

---

### 3. Validation Gate Patterns

#### Pattern: Two-Tier Gates
**Intent**: Combine universal static gates with dynamically generated task-specific gates.

**Tier 1 - Universal (always apply)**:
```yaml
universal_gates:
  semantic_coherence:
    check: "Does conclusion follow from reasoning?"
  completeness:
    check: "Were all requirements addressed?"
  safety:
    check: "Any destructive commands without confirmation?"
```

**Tier 2 - Dynamic (generated per task)**:
```yaml
# Auto-generated for "Fix auth bug" task
task_gates:
  auth_middleware_present:
    type: file_check
    check: "grep -r 'requireAdmin' src/routes/"
  test_coverage:
    type: command
    check: "grep -l 'admin.*401' tests/"
  no_bypass:
    type: llm_review
    check: "Review for auth bypass vulnerabilities"
```

#### Pattern: Gate Generator Meta-Prompt
**Intent**: Agent generates task-specific validation criteria before starting work.

```
You are generating validation gates for: {task_description}

Generate 2-4 checks that are:
1. MEASURABLE (verifiable programmatically or by inspection)
2. SPECIFIC to this exact task (not generic)
3. Include POSITIVE checks (did X happen) and NEGATIVE (did NOT do Y)
4. Consider edge cases unique to this request

Output format:
- name: short identifier
- type: "command" | "llm_review" | "file_check"
- check: the validation
- pass_condition: explicit success criteria
```

#### Pattern: Adversarial Self-Critique
**Intent**: Agent red-teams its own output before presenting to user.

```
[Self-Critique Checklist]
□ Does this actually solve the stated problem?
□ Did I introduce any new bugs?
□ Is there a simpler approach I missed?
□ What could go wrong with this change?
□ Would I approve this if someone else wrote it?
```

---

### 4. Progressive Disclosure Patterns

#### Pattern: Output Styles
**Intent**: Let users control detail level based on their needs.

```
Styles: --concise | --standard | --explanatory | --learning

--concise:    "Fixed. 3 files changed."
--standard:   "Fixed auth bug. Changed middleware.js, routes.js, tests.js"
--explanatory: "Fixed auth bug by adding role check before route handler..."
--learning:   "Here's what I changed and why. Try implementing step 3 yourself..."
```

#### Pattern: Expandable Sections
**Intent**: Show summary by default, allow drill-down on demand.

```
✓ Tests passed (12/12)
  └─ [expand: show test names]

✗ Lint issues (3 warnings)
  └─ [expand: show issues]
  └─ [expand: show fixes]
```

#### Pattern: Capability Discovery
**Intent**: Reveal available tools progressively rather than overwhelming upfront.

```
[Default] Showing 5 most relevant tools for your task
[Available] 12 more tools for advanced workflows - type /tools to see all
[Just discovered] New MCP tool: database-query - useful for performance analysis
```

---

### 5. Interrupt & Resume Patterns

#### Pattern: Explicit Pause Signal
**Intent**: Make it clear when agent is waiting and what input is expected.

```
⏸ PAUSING - Awaiting approval

Review the proposed changes below:
[diff output]

Options:
  [Y] Proceed with changes
  [N] Cancel and explain why
  [E] Edit the proposed changes
  [?] Ask me questions about the changes
```

#### Pattern: Checkpoint Resume
**Intent**: Save state at key points so interrupted work can continue.

```
Resuming task from 2 hours ago:
  ✓ Completed: static analysis (22 issues found)
  ✓ Completed: test coverage (78%)
  → Remaining: security review

Ready to continue? [Y/n]
```

#### Pattern: Graceful Termination
**Intent**: Allow clean shutdown rather than abrupt kill.

```
User: ^C

Agent: Received interrupt. Completing current file write...
       Saving progress checkpoint...
       Summary: 3/5 files processed, checkpoint saved.
       Resume with: agentctl resume task-123
```

---

### 6. Trust Calibration Patterns

#### Pattern: Tiered Autonomy
**Intent**: Different action types require different approval levels.

| Tier | Action Type | Approval |
|------|-------------|----------|
| 0 | Read-only (gather info) | None |
| 1 | Reversible (create file) | Simple confirm |
| 2 | Impactful (modify many files) | Explicit approval |
| 3 | Irreversible (delete, deploy) | Multi-step confirmation |

#### Pattern: Reasoning Visibility
**Intent**: Show thinking process so users can verify reasoning, not just output.

```
[Thinking...]
I considered three approaches:
  A) Direct implementation - fastest but no safety checks
  B) With validation - slight overhead, catches edge cases
  C) With rollback - most defensive, highest overhead

Recommending B because: data deletion context warrants validation,
but rollback overhead isn't justified for this operation.
```

#### Pattern: Uncertainty Communication
**Intent**: Explicitly state when agent doesn't have enough information.

```
I need more context to proceed confidently:

What I know:
  ✓ Function name and location
  ✓ Basic behavior expected

What I don't know:
  ? Performance requirements
  ? Error handling preferences
  ? Integration constraints

Provide these details, or I can proceed with default assumptions
(documented in output).
```

---

## Integration with Leviathan Concepts

### BD (Beads) Integration

lev-ref patterns inform how agents should communicate about issue tracking:

```
[bd integration example]
Found 3 related issues during analysis:
  - lev-123: Auth middleware needs refactor (IN_PROGRESS)
  - lev-456: Add role-based access control (BLOCKED by lev-123)
  - lev-789: Security audit findings (OPEN)

Should I:
  A) Focus only on current task, ignore related issues
  B) Update lev-123 as I work and note connections
  C) Create new issue for discovered scope
```

### Roadmap Alignment

Patterns support roadmap visibility:

```
[Roadmap context]
This task touches: Agent/PM Layer → Command Consolidation
Related work: h-poly-bridge-architecture (Task 03 waits for this)

Impact: Completing this unblocks downstream work.
Confidence: High - clear requirements, tested approach.
```

---

## Adding New Patterns

When discovering new agentic UX patterns:

1. **Document the intent** - What problem does this solve?
2. **Show good/bad examples** - Concrete before/after
3. **Describe implementation** - How to apply it
4. **Tag the source** - Where did this pattern come from?
5. **Add to bd** - `bd create "New pattern: {name}" --type=task --priority=3`

---

## Sources

Research synthesis from:
- Meta-Prompting Protocol (arxiv 2512.15053) - TextGrad backpropagation
- IntuitionLabs Meta-Prompting - Self-critique patterns
- Anthropic Claude Code Best Practices - Plan-then-act
- LangGraph Documentation - Interrupt and checkpoint patterns
- JetBrains AI Blog - Trust calibration
- Production CLI agents: Aider, Cursor, GitHub Copilot CLI

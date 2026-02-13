# Stage 5: Interaction Models
**Lev UI Workspace Runtime**

**Version:** 1.0
**Date:** 2026-02-02
**Status:** Spec Phase
**Dependencies:** .beads/types/*.yaml, lev-ui-prd.md

---

## Executive Summary

This specification maps all Beads entity FSMs to UI state machines, defines the 4D layout space permutation system, specifies intent classification for cognitive layer routing, and establishes notification routing rules. All specifications reference actual entity types from `.beads/types/*.yaml` and existing daemon/dashboard lifecycles.

**Key Insight:** The UI is a **reactive projection** of entity state. Every visual element corresponds to an entity FSM state. Layout adapts to `(cognitive_layer, entity_focus, interaction_mode, density)`.

---

## 1. Entity FSM → UI State Machine Mapping

### 1.1 Session Entity

**Source:** `.beads/types/session.yaml`

```
┌──────────────────────────────────────────────────────────────┐
│ SESSION LIFECYCLE                                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [new] → [active] → [checkpoint] → [crystallize] → [closed] │
│           ↓           ↓                                      │
│        [idle]      [archived]                                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**FSM States:**
- `new` → Freshly created, no work done yet
- `active` → User/agent actively working
- `idle` → No recent activity (>30min since last update)
- `checkpoint` → User saved interim state (bd://comments)
- `crystallize` → Extracting ideas (prompt://idea-extractor)
- `closed` → Session ended, archived
- `archived` → Historical reference only

**UI State Mapping:**

| FSM State | Visual Representation | Icon | Color | Actions Available | Notifications |
|-----------|----------------------|------|-------|-------------------|---------------|
| `new` | Badge: "NEW" | ○ | Gray | [Start Work], [Delete] | None |
| `active` | Pulsing indicator | ● | Green | [Checkpoint], [Crystallize], [Close] | Auto-save every 2min |
| `idle` | Dim text, "Last active Xmin ago" | ○ | Yellow | [Resume], [Close], [Archive] | Reminder after 1hr: "Resume or close?" |
| `checkpoint` | Badge: "CHECKPOINT" | ◐ | Blue | [Resume], [View Checkpoint], [Crystallize] | None |
| `crystallize` | Loading spinner | ⟳ | Purple | [View Progress], [Cancel] | On complete: "3 ideas extracted" |
| `closed` | Strikethrough text | ✓ | Gray | [Reopen], [Archive] | None |
| `archived` | Collapsed by default | □ | Light gray | [View], [Restore] | None |

**Transition Triggers:**

| Transition | User-Initiated | System-Initiated | Action |
|-----------|----------------|------------------|--------|
| `new → active` | Click "Start Work" | Agent begins execution | Open canvas (terminal/chat) |
| `active → idle` | - | No updates for 30min | Show idle badge, dim UI |
| `active → checkpoint` | Click "Checkpoint" | - | Trigger `bd://comments`, save state |
| `active → crystallize` | Click "Crystallize" | - | Invoke `prompt://idea-extractor` |
| `crystallize → closed` | - | Extractor completes | Show "N ideas extracted" modal |
| `idle → active` | Click "Resume" | Agent resumes | Re-open canvas at last position |
| `closed → archived` | Click "Archive" | Auto after 7 days | Move to archive panel |

**Notifications:**

- **Auto-save (active)**: Silent background save, no UI notification
- **Idle warning**: Toast after 1hr: "Session 'X' idle for 1hr. Resume or close?"
- **Crystallization complete**: Modal: "Extracted 3 ideas from session 'X'. View?"
- **Checkpoint saved**: Toast: "Checkpoint saved at 14:32"

---

### 1.2 Idea Entity

**Source:** `.beads/types/idea.yaml`

```
┌──────────────────────────────────────────────────────────────┐
│ IDEA LIFECYCLE                                               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [ideation] → [analysis] → [specification] → [implementation]│
│                                ↓                   ↓         │
│                          [production]         [archived]     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**FSM States:**
- `ideation` → Initial concept capture
- `analysis` → Research and validation
- `specification` → Formal design
- `implementation` → Active development (workshop/forge)
- `production` → Released and maintained
- `archived` → Historical reference

**UI State Mapping:**

| FSM State | Visual Representation | Icon | Color | Actions Available | Notifications |
|-----------|----------------------|------|-------|-------------------|---------------|
| `ideation` | Badge: "IDEA" | 💡 | Yellow | [Analyze], [Archive] | None |
| `analysis` | Progress bar (research %) | 🔬 | Blue | [View Research], [Specify], [Archive] | On confidence >85%: "Ready for specification" |
| `specification` | Badge: "SPEC" | 📐 | Purple | [View Spec], [Implement], [Archive] | None |
| `implementation` | Progress bar (tasks %) | ⚡ | Orange | [View Tasks], [View Workshop], [Ship] | On all tasks done: "Ready for production" |
| `production` | Badge: "LIVE" | ✅ | Green | [Monitor], [Archive] | None |
| `archived` | Collapsed by default | 📦 | Gray | [View], [Restore] | None |

**Transition Triggers:**

| Transition | User-Initiated | System-Initiated | Action |
|-----------|----------------|------------------|--------|
| `ideation → analysis` | Click "Analyze" | - | Create research session |
| `analysis → specification` | Click "Specify" | Confidence >85% | Create spec document |
| `specification → implementation` | Click "Implement" | - | Create workshop tasks |
| `implementation → production` | Click "Ship" | All tasks completed | Deploy to production |
| `* → archived` | Click "Archive" | - | Move to archives |

**Trajectory Markers (Visual Tags):**

- **Ephemeral**: Badge "EPH" (needs periodic revisit)
- **Workshop**: Badge "WIP" (active in forge)
- **Memories**: Badge "MEM" (archived permanent reference)

**Notifications:**

- **Analysis threshold**: Toast: "Idea 'X' reached 85% confidence. Ready to specify?"
- **Implementation complete**: Modal: "All tasks complete for 'X'. Ship to production?"
- **Workshop activity**: Badge count on idea: "(3 active tasks)"

---

### 1.3 Synth Entity

**Source:** `.beads/types/synth.yaml`

```
┌──────────────────────────────────────────────────────────────┐
│ SYNTH LIFECYCLE                                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [gap_detected] → [created] → [active] → [successful]       │
│                                  ↓           ↓               │
│                              [failed]   [promoted]           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**FSM States:**
- `gap_detected` → CEO confidence <70%, gap analysis triggered
- `created` → Synth initialized with role/job
- `active` → Synth being used for thinking
- `successful` → Task completed successfully, metrics updated
- `failed` → Task failed, confidence adjusted
- `promoted` → Auto-promoted to skill (usage >5, success >80%)

**UI State Mapping:**

| FSM State | Visual Representation | Icon | Color | Actions Available | Notifications |
|-----------|----------------------|------|-------|-------------------|---------------|
| `gap_detected` | Toast: "Gap in {area}" | ⚠️ | Orange | [Create Synth], [Ignore] | Gap analysis result |
| `created` | Badge: "SYNTH" | 🧠 | Blue | [Use], [Edit Role], [Delete] | None |
| `active` | Pulsing indicator | ⟳ | Green | [View Context], [Cancel] | None |
| `successful` | +1 success count | ✓ | Green | [Use Again], [View Metrics] | On 5th success: "Eligible for promotion" |
| `failed` | Badge: "FAILED" | ✗ | Red | [Retry], [Adjust], [Delete] | Failure reason logged |
| `promoted` | Badge: "SKILL" | ⭐ | Gold | [View Skill File] | Modal: "Synth 'X' promoted to skill!" |

**Metrics Display (Always Visible):**

```
┌────────────────────────────────────────┐
│ Synth: "Research Specialist"           │
│ Role: Deep domain analysis             │
│ Job: Systematic literature review      │
│                                        │
│ Metrics:                               │
│  Usage: 7 times                        │
│  Success: 85.7% (6/7)                  │
│  Confidence: 82%                       │
│  Breakthroughs: 2                      │
│                                        │
│ [Use Synth] [View History] [Promote]  │
└────────────────────────────────────────┘
```

**Transition Triggers:**

| Transition | User-Initiated | System-Initiated | Action |
|-----------|----------------|------------------|--------|
| `gap_detected → created` | Click "Create Synth" | - | Open synth creator wizard |
| `created → active` | Click "Use Synth" | Agent invokes synth | Show synth working indicator |
| `active → successful` | - | Synth completes task | Increment metrics, update confidence |
| `active → failed` | - | Synth fails | Log failure, adjust confidence |
| `successful → promoted` | Click "Promote" | Usage >5 & success >80% | Generate skill file, archive synth |

**Auto-Promotion Rules:**

```yaml
promotion_trigger:
  conditions:
    - usage_count >= 5
    - success_rate >= 0.8
  notification: "Synth '{role}' eligible for promotion to skill. Promote now?"
  action: skill://codifier
  target: ~/.claude/skills/{synth-role-slug}.md
```

**Notifications:**

- **Gap detected**: Toast: "Low confidence in {area}. Create synth?"
- **Success milestone**: Toast: "Synth '{role}' used 5 times with 80% success. Promote?"
- **Promotion complete**: Modal: "Synth promoted to skill: ~/.claude/skills/{name}.md"
- **Failure**: Toast: "Synth '{role}' failed: {reason}. Retry?"

---

### 1.4 Artifact Entity

**Source:** `.beads/types/artifact.yaml`

```
┌──────────────────────────────────────────────────────────────┐
│ ARTIFACT LIFECYCLE                                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [created] → [valid] → [accessed] → [stale] → [orphaned]    │
│                ↓                                             │
│           [invalid]                                          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**FSM States:**
- `created` → Artifact registered with external_ref
- `valid` → External ref validated (file exists, URL reachable, etc.)
- `accessed` → User/agent accessed the artifact
- `stale` → Parent session closed, artifact orphaned
- `invalid` → External ref no longer valid (file deleted, URL 404, etc.)
- `orphaned` → Parent entity archived, artifact needs migration

**UI State Mapping:**

| FSM State | Visual Representation | Icon | Color | Actions Available | Notifications |
|-----------|----------------------|------|-------|-------------------|---------------|
| `created` | Badge: "NEW" | 📄 | Gray | [Validate], [Edit], [Delete] | None |
| `valid` | Link to external_ref | 📎 | Blue | [Open], [Copy Path], [Unlink] | None |
| `accessed` | Badge: "ACCESSED" | 👁️ | Green | [Open], [View History] | None |
| `stale` | Badge: "STALE" | ⏰ | Orange | [Migrate], [Archive], [Delete] | Warning: "Parent session closed" |
| `invalid` | Badge: "BROKEN" | ⚠️ | Red | [Fix Ref], [Delete] | Error: "External ref invalid" |
| `orphaned` | Badge: "ORPHAN" | 🔗 | Yellow | [Re-link], [Archive] | Prompt: "Migrate to new parent?" |

**External Ref Protocol Icons:**

| Protocol | Icon | Link Behavior |
|----------|------|---------------|
| `file://` | 📁 | Open in default editor/viewer |
| `https://` | 🌐 | Open in browser |
| `bd://` | 🎯 | Navigate to BD issue |
| `manifest://` | ⚙️ | Execute manifestation plugin |
| `git://` | 🔀 | Show git object (commit, file) |
| `skill://` | 🧩 | Invoke skill tool |

**Transition Triggers:**

| Transition | User-Initiated | System-Initiated | Action |
|-----------|----------------|------------------|--------|
| `created → valid` | - | Auto on create | Validate external_ref protocol |
| `valid → accessed` | Click "Open" | Agent accesses ref | Open external ref, log access |
| `valid → invalid` | - | Ref validation fails | Show broken link indicator |
| `accessed → stale` | - | Parent session closes | Show stale badge |
| `stale → orphaned` | - | Parent archived | Prompt migration |

**Notifications:**

- **Invalid ref**: Toast: "Artifact '{title}' ref is invalid: {reason}"
- **Orphaned**: Modal: "Artifact '{title}' orphaned. Migrate to new parent or archive?"
- **Stale**: Badge on artifact + tooltip: "Parent session closed Xd ago"

---

### 1.5 Memory Entity

**Source:** `.beads/types/memory.yaml`

```
┌──────────────────────────────────────────────────────────────┐
│ MEMORY LIFECYCLE                                             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [extracted] → [ephemeral] → [validated] → [permanent]      │
│                     ↓                                        │
│                 [decayed]                                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**FSM States:**
- `extracted` → Agent extracted pattern from conversation
- `ephemeral` → Low confidence (<0.85), auto-decay candidate
- `validated` → Confidence ≥0.85, reused successfully
- `permanent` → Promoted to consolidated knowledge
- `decayed` → Not reused, confidence dropped, auto-archived

**UI State Mapping:**

| FSM State | Visual Representation | Icon | Color | Actions Available | Notifications |
|-----------|----------------------|------|-------|-------------------|---------------|
| `extracted` | Badge: "MEM" | 🧠 | Blue | [View], [Promote], [Archive] | None |
| `ephemeral` | Badge: "EPH" + decay timer | ⏳ | Yellow | [Validate], [Archive] | Warning: "Decays in Xd" |
| `validated` | Badge: "VAL" | ✓ | Green | [Promote], [View Uses] | None |
| `permanent` | Badge: "PERM" | ⭐ | Purple | [View], [Edit] | None |
| `decayed` | Strikethrough text | 💀 | Gray | [Restore], [Delete] | None |

**Confidence Display:**

```
┌────────────────────────────────────────┐
│ Memory: "Port collision prevention"    │
│ Pattern: Always allocate from pool     │
│ Confidence: 92%                        │
│ Reused: 7 times                        │
│ Origin: session:abc123                 │
│ Labels: [-l perm]                      │
│                                        │
│ [Apply Pattern] [View Context]        │
└────────────────────────────────────────┘
```

**Transition Triggers:**

| Transition | User-Initiated | System-Initiated | Action |
|-----------|----------------|------------------|--------|
| `extracted → ephemeral` | - | Confidence <0.85 | Set decay timer (7d) |
| `extracted → permanent` | Click "Promote" | Confidence ≥0.85 | Add `-l perm` label |
| `ephemeral → validated` | - | Reused successfully | Increase confidence |
| `ephemeral → decayed` | - | Not reused in 7d | Auto-archive |
| `validated → permanent` | Click "Promote" | Confidence >90% | Consolidate to knowledge base |

**Notifications:**

- **Decay warning**: Toast: "Memory '{pattern}' decays in 2d. Validate or archive?"
- **Promotion eligible**: Toast: "Memory '{pattern}' reused 5 times. Promote to permanent?"
- **Decayed**: Silent auto-archive, no notification

---

### 1.6 Message Entity

**Source:** `.beads/types/message.yaml`

```
┌──────────────────────────────────────────────────────────────┐
│ MESSAGE LIFECYCLE                                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [sent] → [delivered] → [read] → [acknowledged] → [closed]  │
│             ↓                                                │
│         [failed]                                             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**FSM States:**
- `sent` → Message created and sent to recipient
- `delivered` → Recipient inbox received message
- `read` → Recipient opened message
- `acknowledged` → Recipient acknowledged/replied
- `failed` → Delivery failed (recipient not found, etc.)
- `closed` → Message archived

**UI State Mapping:**

| FSM State | Visual Representation | Icon | Color | Actions Available | Notifications |
|-----------|----------------------|------|-------|-------------------|---------------|
| `sent` | Badge: "SENT" | ✉️ | Blue | [View], [Cancel] | None |
| `delivered` | Badge: "DELIVERED" | 📬 | Green | [View], [Archive] | None |
| `read` | Badge: "READ" | 👁️ | Purple | [View], [Archive] | None |
| `acknowledged` | Badge: "ACK" | ✓ | Green | [View Thread], [Archive] | None |
| `failed` | Badge: "FAILED" | ⚠️ | Red | [Retry], [Delete] | Error: "Delivery failed: {reason}" |
| `closed` | Collapsed by default | □ | Gray | [View] | None |

**Priority Indicators:**

- **Urgent**: Red pulsing border + sound alert
- **High**: Orange border
- **Normal**: No special indicator
- **Low**: Dim text

**Transition Triggers:**

| Transition | User-Initiated | System-Initiated | Action |
|-----------|----------------|------------------|--------|
| `sent → delivered` | - | Recipient inbox processes | Update badge |
| `delivered → read` | - | Recipient opens message | Update badge + timestamp |
| `read → acknowledged` | Recipient replies | - | Link reply to original |
| `sent → failed` | - | Delivery error | Show error notification |
| `* → closed` | Click "Archive" | Auto after 30d | Move to closed messages |

**Notifications:**

- **New message**: Toast: "New message from {sender}: {preview}"
- **Urgent message**: Modal + sound: "URGENT from {sender}: {content}"
- **Delivery failed**: Toast: "Message to {recipient} failed: {reason}"
- **Acknowledged**: Toast: "{recipient} acknowledged your message"

---

### 1.7 Capability Entity

**Source:** `.beads/types/capability.yaml`

```
┌──────────────────────────────────────────────────────────────┐
│ CAPABILITY LIFECYCLE                                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [draft] → [specified] → [tested] → [implemented] → [released]│
│              ↓                                               │
│          [blocked]                                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**FSM States:**
- `draft` → Initial user story capture
- `specified` → Acceptance criteria defined (Given/When/Then)
- `tested` → Test skeleton created (BDD → TDD)
- `implemented` → Code complete, tests passing
- `released` → Shipped to production
- `blocked` → Dependency or blocker preventing progress

**UI State Mapping:**

| FSM State | Visual Representation | Icon | Color | Actions Available | Notifications |
|-----------|----------------------|------|-------|-------------------|---------------|
| `draft` | Badge: "DRAFT" | 📝 | Gray | [Specify], [Delete] | None |
| `specified` | Badge: "SPEC" | 📐 | Blue | [Create Tests], [View Spec] | None |
| `tested` | Progress bar (test coverage %) | 🧪 | Orange | [Implement], [View Tests] | On coverage <80%: "Add tests" |
| `implemented` | Badge: "DONE" | ✅ | Green | [Release], [View Code] | On tests fail: "Fix failing tests" |
| `released` | Badge: "LIVE" | 🚀 | Purple | [Monitor], [Archive] | None |
| `blocked` | Badge: "BLOCKED" | 🚫 | Red | [Resolve Blocker], [Reassign] | Alert: "Blocked by {reason}" |

**BDD Contract Display:**

```
┌────────────────────────────────────────┐
│ Capability: "User Login"               │
│                                        │
│ User Story:                            │
│ As a user                              │
│ I want to log in with email/password  │
│ So that I can access my account        │
│                                        │
│ Acceptance Criteria:                   │
│ GIVEN user has valid credentials       │
│ WHEN user submits login form           │
│ THEN user is redirected to dashboard   │
│                                        │
│ Tests: 12 scenarios, 85% coverage      │
│ [View Scenarios] [Run Tests]          │
└────────────────────────────────────────┘
```

**Transition Triggers:**

| Transition | User-Initiated | System-Initiated | Action |
|-----------|----------------|------------------|--------|
| `draft → specified` | Click "Specify" | - | Invoke BDD template generator |
| `specified → tested` | Click "Create Tests" | - | Generate test skeleton from scenarios |
| `tested → implemented` | - | All tests passing | Mark as implemented |
| `implemented → released` | Click "Release" | - | Deploy to production |
| `* → blocked` | Click "Block" | Dependency fails | Show blocker modal |

**Notifications:**

- **Coverage low**: Toast: "Test coverage 65% (target 80%). Add scenarios?"
- **Tests failing**: Alert: "3 tests failing. Fix before release?"
- **Ready for release**: Modal: "All tests passing. Release to production?"
- **Blocked**: Alert: "Blocked by {dependency}. Resolve blocker?"

---

### 1.8 Initiative Entity

**Source:** `.beads/types/initiative.yaml`

```
┌──────────────────────────────────────────────────────────────┐
│ INITIATIVE LIFECYCLE                                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [draft] → [approved] → [in_progress] → [completed]         │
│                            ↓                                 │
│                        [blocked]                             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**FSM States:**
- `draft` → Vision and scope defined
- `approved` → Stakeholders approved, ready to start
- `in_progress` → Active work on capabilities/epics
- `blocked` → Critical blocker preventing progress
- `completed` → All capabilities delivered, KPIs met

**UI State Mapping:**

| FSM State | Visual Representation | Icon | Color | Actions Available | Notifications |
|-----------|----------------------|------|-------|-------------------|---------------|
| `draft` | Badge: "DRAFT" | 📋 | Gray | [Edit], [Approve], [Delete] | None |
| `approved` | Badge: "APPROVED" | ✓ | Blue | [Start], [View Plan] | None |
| `in_progress` | Progress bar (completion %) | ⚡ | Orange | [View Specs], [Monitor KPIs] | Milestone reached |
| `blocked` | Badge: "BLOCKED" | 🚫 | Red | [Resolve], [Escalate] | Alert: "Critical blocker" |
| `completed` | Badge: "COMPLETE" | 🎉 | Green | [Archive], [View Results] | None |

**Hierarchy Display (L0 View):**

```
┌────────────────────────────────────────┐
│ Initiative: "Auth System Refactor"     │
│ Vision: Modernize authentication       │
│ Progress: 65% (13/20 capabilities)     │
│                                        │
│ Capabilities:                          │
│  ✅ User Login (released)              │
│  ✅ Password Reset (released)          │
│  ⚡ OAuth Integration (in progress)    │
│  📝 2FA (draft)                        │
│  ... +16 more                          │
│                                        │
│ KPIs:                                  │
│  - Auth latency: 120ms (target 100ms)  │
│  - Test coverage: 92% (target 90%)     │
│                                        │
│ [View All Specs] [Monitor]            │
└────────────────────────────────────────┘
```

**Transition Triggers:**

| Transition | User-Initiated | System-Initiated | Action |
|-----------|----------------|------------------|--------|
| `draft → approved` | Click "Approve" | - | Lock scope, notify team |
| `approved → in_progress` | Click "Start" | - | Create first capabilities |
| `in_progress → blocked` | Click "Block" | Critical blocker detected | Escalate to owner |
| `in_progress → completed` | - | All specs closed + KPIs met | Trigger completion flow |

**Notifications:**

- **Milestone reached**: Toast: "Initiative '{vision}' 50% complete!"
- **KPI warning**: Alert: "Auth latency 150ms (target 100ms). Investigate?"
- **All specs done**: Modal: "All capabilities complete. Mark initiative done?"
- **Blocked**: Alert: "Initiative blocked: {reason}. Escalate?"

---

### 1.9 Proposal Entity

**Source:** `.beads/types/proposal.yaml`

```
┌──────────────────────────────────────────────────────────────┐
│ PROPOSAL LIFECYCLE                                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [draft] → [submitted] → [under_review] → [approved]        │
│                                ↓              ↓              │
│                           [rejected]     [deferred]          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**FSM States:**
- `draft` → Problem and solution being drafted
- `submitted` → Submitted for review
- `under_review` → Stakeholders reviewing
- `approved` → Decision to proceed
- `rejected` → Decision not to proceed
- `deferred` → Decision postponed, revisit later

**UI State Mapping:**

| FSM State | Visual Representation | Icon | Color | Actions Available | Notifications |
|-----------|----------------------|------|-------|-------------------|---------------|
| `draft` | Badge: "DRAFT" | 📝 | Gray | [Edit], [Submit] | None |
| `submitted` | Badge: "SUBMITTED" | 📤 | Blue | [View], [Withdraw] | Stakeholders notified |
| `under_review` | Badge: "REVIEW" + reviewer list | 👀 | Orange | [View Comments], [Update] | Reviewers comment |
| `approved` | Badge: "APPROVED" | ✅ | Green | [Create Plan], [View Decision] | Decision logged |
| `rejected` | Badge: "REJECTED" | ❌ | Red | [Revise], [Archive] | Rationale captured |
| `deferred` | Badge: "DEFERRED" + revisit date | ⏸️ | Yellow | [Revisit], [Archive] | Reminder on revisit date |

**Decision Log Display:**

```
┌────────────────────────────────────────┐
│ Proposal: "Migrate to PostgreSQL"      │
│                                        │
│ Problem:                               │
│ SQLite doesn't scale for multi-user    │
│                                        │
│ Proposed Solution:                     │
│ Migrate to PostgreSQL cluster          │
│                                        │
│ Alternatives Considered:               │
│ - MySQL (rejected: licensing)          │
│ - MongoDB (rejected: schema flexibility)│
│                                        │
│ Decision Log:                          │
│ 2026-01-15: Submitted                  │
│ 2026-01-20: Under review (3 reviewers) │
│ 2026-01-25: Approved by PM             │
│                                        │
│ [Create Implementation Plan]          │
└────────────────────────────────────────┘
```

**Transition Triggers:**

| Transition | User-Initiated | System-Initiated | Action |
|-----------|----------------|------------------|--------|
| `draft → submitted` | Click "Submit" | - | Notify stakeholders |
| `submitted → under_review` | Reviewer starts review | - | Show reviewer badges |
| `under_review → approved` | Reviewer approves | All reviewers approve | Log decision, create plan |
| `under_review → rejected` | Reviewer rejects | - | Log decision, capture rationale |
| `under_review → deferred` | Reviewer defers | - | Set revisit date |
| `deferred → under_review` | - | Revisit date reached | Notify stakeholders |

**Notifications:**

- **Submitted**: Toast to stakeholders: "New proposal: '{title}'. Review?"
- **Comment added**: Toast: "{reviewer} commented on proposal '{title}'"
- **Decision made**: Modal: "Proposal '{title}' {approved|rejected|deferred}. {rationale}"
- **Revisit reminder**: Toast: "Deferred proposal '{title}' ready for revisit"

---

### 1.10 Plan Entity

**Source:** `.beads/types/plan.yaml`

```
┌──────────────────────────────────────────────────────────────┐
│ PLAN LIFECYCLE                                               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [draft] → [approved] → [in_progress] → [completed]         │
│                             ↓              ↓                 │
│                        [blocked]      [cancelled]            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**FSM States:**
- `draft` → Objective and phases being drafted
- `approved` → Ready to execute
- `in_progress` → Tasks being worked on
- `blocked` → Critical blocker preventing progress
- `completed` → Success criteria met
- `cancelled` → Cancelled before completion

**UI State Mapping:**

| FSM State | Visual Representation | Icon | Color | Actions Available | Notifications |
|-----------|----------------------|------|-------|-------------------|---------------|
| `draft` | Badge: "DRAFT" | 📋 | Gray | [Edit], [Approve] | None |
| `approved` | Badge: "APPROVED" | ✓ | Blue | [Start], [View Phases] | None |
| `in_progress` | Progress bar (phase completion %) | ⚡ | Orange | [View Tasks], [Block], [Complete] | Phase milestones |
| `blocked` | Badge: "BLOCKED" | 🚫 | Red | [Unblock], [Cancel] | Blocker logged |
| `completed` | Badge: "COMPLETE" | ✅ | Green | [Retrospective], [Archive] | Success criteria met |
| `cancelled` | Badge: "CANCELLED" | ❌ | Gray | [View Reason], [Revive] | Cancellation logged |

**Phase Progress Display:**

```
┌────────────────────────────────────────┐
│ Plan: "PostgreSQL Migration"           │
│ Objective: Migrate from SQLite         │
│ Progress: Phase 2/4 (50%)              │
│                                        │
│ Phases:                                │
│  ✅ Phase 1: Schema design (done)      │
│  ⚡ Phase 2: Data migration (70%)      │
│  📋 Phase 3: App refactor (pending)    │
│  📋 Phase 4: Production cutover (pending)│
│                                        │
│ Success Criteria:                      │
│  ✅ Zero data loss                     │
│  ⏳ <5min downtime (target)            │
│  ⏳ All tests passing (92%)            │
│                                        │
│ [View Tasks] [Unblock] [Retrospective]│
└────────────────────────────────────────┘
```

**Transition Triggers:**

| Transition | User-Initiated | System-Initiated | Action |
|-----------|----------------|------------------|--------|
| `draft → approved` | Click "Approve" | - | Validate dependencies |
| `approved → in_progress` | Click "Start" | - | Create initial tasks |
| `in_progress → blocked` | Click "Block" | Blocker detected | Log blocker, escalate |
| `in_progress → completed` | Click "Complete" | All tasks done + criteria met | Prompt retrospective |
| `in_progress → cancelled` | Click "Cancel" | - | Log cancellation reason |

**Notifications:**

- **Phase complete**: Toast: "Phase '{name}' complete! Starting phase '{next}'"
- **Blocked**: Alert: "Plan blocked: {reason}. Escalate to owner?"
- **Success criteria met**: Modal: "All criteria met. Mark plan complete?"
- **Retrospective prompt**: Modal: "Plan complete. Create retrospective?"

---

### 1.11 Daemon State Machine

**Source:** `~/lev/core/daemon/src/daemon-core.ts`

```
┌──────────────────────────────────────────────────────────────┐
│ DAEMON LIFECYCLE                                             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [stopped] → [starting] → [running] → [stopping] → [stopped]│
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**FSM States:**
- `stopped` → No daemon process running
- `starting` → Initializing (PID file written, timers not started)
- `running` → Heartbeat, queue poll, BD sync active
- `stopping` → Gracefully shutting down (clearing timers, tasks)

**UI State Mapping:**

| FSM State | Visual Representation | Icon | Color | Actions Available | Notifications |
|-----------|----------------------|------|-------|-------------------|---------------|
| `stopped` | Badge: "OFFLINE" | ○ | Gray | [Start Daemon] | None |
| `starting` | Loading spinner | ⟳ | Blue | [View Logs] | None |
| `running` | Pulsing green dot + uptime | ● | Green | [Stop], [View Status], [View Queue] | Heartbeat every 30s |
| `stopping` | Loading spinner | ⟳ | Orange | [View Logs] | None |

**Daemon Bubble Display (Per Project):**

```
┌────────────────────────────────────────┐
│ Daemon: leviathan                      │
│ Status: RUNNING                        │
│ Uptime: 2h 34m                         │
│                                        │
│ Processes:                             │
│  ● index backend (port 9200, pid 1234) │
│  ● poly-runner (port 8080, pid 1235)   │
│  ● dashboard (port 3001, pid 1236)     │
│                                        │
│ Queue: 3 tasks pending, 1 active       │
│ Last BD Sync: 12s ago                  │
│                                        │
│ [Stop Daemon] [View Logs] [Restart]   │
└────────────────────────────────────────┘
```

**Heartbeat Events:**

```typescript
// Every 30s, daemon emits heartbeat event
{
  event: 'daemon.heartbeat',
  pid: 12345,
  uptime: 9240000, // ms
  activeTasks: 1,
  queueStatus: { pending: 3, active: 1, completed: 12 }
}
```

**Transition Triggers:**

| Transition | User-Initiated | System-Initiated | Action |
|-----------|----------------|------------------|--------|
| `stopped → starting` | Click "Start Daemon" | - | Initialize daemon, write PID file |
| `starting → running` | - | Timers started | Show running badge |
| `running → stopping` | Click "Stop" | SIGTERM received | Clear timers, finish active tasks |
| `stopping → stopped` | - | Cleanup complete | Remove PID file |

**Notifications:**

- **Started**: Toast: "Daemon started (pid {pid})"
- **Heartbeat failure**: Alert: "Daemon heartbeat missed. Check logs?"
- **Stopped**: Toast: "Daemon stopped. {tasksProcessed} tasks processed."
- **Crash**: Alert: "Daemon crashed! Restart?"

---

### 1.12 Dashboard State Machine

**Source:** `~/lev/core/agent-harness/vendor/AgentPing/packages/dashboard-runner/src/runner.ts`

```
┌──────────────────────────────────────────────────────────────┐
│ DASHBOARD LIFECYCLE                                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [stopped] → [starting] → [online] → [stopping] → [stopped] │
│                             ↓                                │
│                         [failed]                             │
│                             ↓                                │
│                       [restarting]                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**FSM States:**
- `stopped` → Dashboard process not running
- `starting` → Process spawning
- `online` → Process running, health check passing
- `failed` → Process crashed or health check failed
- `restarting` → Attempting restart after failure
- `stopping` → Gracefully shutting down

**UI State Mapping:**

| FSM State | Visual Representation | Icon | Color | Actions Available | Notifications |
|-----------|----------------------|------|-------|-------------------|---------------|
| `stopped` | Badge: "OFFLINE" | ○ | Gray | [Start] | None |
| `starting` | Loading spinner + "Starting..." | ⟳ | Blue | [View Logs] | None |
| `online` | Badge: "ONLINE" + health indicator | ● | Green | [Stop], [Restart], [Open Dashboard] | None |
| `failed` | Badge: "FAILED" + crash count | ✗ | Red | [View Logs], [Restart] | Alert: "Dashboard crashed" |
| `restarting` | Loading spinner + "Restarting..." | ⟳ | Orange | [View Logs], [Cancel] | None |
| `stopping` | Loading spinner + "Stopping..." | ⟳ | Orange | [View Logs] | None |

**Dashboard Card Display:**

```
┌────────────────────────────────────────┐
│ Dashboard: jarvis-voice                │
│ Status: ONLINE                         │
│ Port: 3001                             │
│ PID: 5678                              │
│ Uptime: 1h 12m                         │
│ Health: ✅ Healthy (last check: 5s ago)│
│ Crashes: 0                             │
│                                        │
│ [Open Dashboard] [Restart] [Stop]     │
└────────────────────────────────────────┘
```

**Health Check Display:**

```yaml
health_check:
  endpoint: http://localhost:3001/health
  interval_ms: 10000
  timeout_ms: 5000
  retry_limit: 3

# UI shows:
# ✅ Healthy (response <500ms)
# ⚠️ Degraded (response 500-2000ms)
# ❌ Failed (timeout or error)
```

**Transition Triggers:**

| Transition | User-Initiated | System-Initiated | Action |
|-----------|----------------|------------------|--------|
| `stopped → starting` | Click "Start" | - | Spawn process |
| `starting → online` | - | Process started + health check passes | Show online badge |
| `online → failed` | - | Process crashes or health check fails | Increment crash count |
| `failed → restarting` | Click "Restart" | Auto-restart (if enabled) | Attempt restart |
| `restarting → online` | - | Process restarted successfully | Reset crash count if >3 |
| `online → stopping` | Click "Stop" | - | Send SIGTERM |
| `stopping → stopped` | - | Process exited | Clear runtime data |

**Notifications:**

- **Started**: Toast: "Dashboard '{id}' started on port {port}"
- **Health check failed**: Alert: "Dashboard '{id}' health check failed ({count}/3)"
- **Crashed**: Alert: "Dashboard '{id}' crashed! Restart?"
- **Auto-restart**: Toast: "Dashboard '{id}' auto-restarting (attempt {n})"
- **Restart failed**: Alert: "Dashboard '{id}' restart failed after 3 attempts"

---

## 2. Workspace State Machines

### 2.1 Panel State Machine

```
┌──────────────────────────────────────────────────────────────┐
│ PANEL LIFECYCLE                                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [collapsed] ⟷ [expanded] → [focused] → [detached] → [docked]│
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**States:**
- `collapsed` → Panel hidden, only icon/badge visible
- `expanded` → Panel visible with content
- `focused` → Panel has keyboard focus, other panels dimmed
- `detached` → Panel floating in separate window
- `docked` → Panel re-attached to main workspace

**Transitions:**

| From | To | Trigger | Action |
|------|----|---------| -------|
| `collapsed` | `expanded` | Click panel icon | Animate expand, load content |
| `expanded` | `collapsed` | Click collapse button | Animate collapse, preserve state |
| `expanded` | `focused` | Click inside panel | Dim other panels, highlight border |
| `focused` | `expanded` | Click outside panel | Remove focus highlight |
| `expanded` | `detached` | Drag panel header | Create floating window |
| `detached` | `docked` | Click "Dock" button | Re-attach to sidebar |

**Visual States:**

| State | Width | Opacity | Border | Z-Index |
|-------|-------|---------|--------|---------|
| `collapsed` | 48px (icon only) | 0.7 | None | 1 |
| `expanded` | 300px | 1.0 | 1px gray | 2 |
| `focused` | 300px | 1.0 | 2px blue | 3 |
| `detached` | User-resizable | 1.0 | Window chrome | 999 |

**Keyboard Shortcuts:**

- `Cmd+B`: Toggle left panel (entity tree)
- `Cmd+Shift+B`: Toggle right panel (daemon bubbles)
- `Cmd+J`: Toggle bottom panel (logs)
- `Esc`: Collapse all panels (focus canvas)

---

### 2.2 Tab State Machine

```
┌──────────────────────────────────────────────────────────────┐
│ TAB LIFECYCLE                                                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [closed] → [open] → [active] ⟷ [backgrounded] → [pinned]   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**States:**
- `closed` → Tab does not exist
- `open` → Tab exists but not visible
- `active` → Tab visible in canvas
- `backgrounded` → Tab exists but another tab is active
- `pinned` → Tab persists across sessions

**Transitions:**

| From | To | Trigger | Action |
|------|----|---------| -------|
| `closed` | `open` | Open entity (agent, session, terminal) | Create tab, set as active |
| `open` | `active` | Click tab | Switch canvas to this tab |
| `active` | `backgrounded` | Click another tab | Hide canvas, preserve state |
| `backgrounded` | `active` | Click tab | Restore canvas state |
| `active` | `pinned` | Click pin icon | Persist tab across sessions |
| `pinned` | `active` | Session restore | Re-open tab on workspace load |
| `* ` | `closed` | Click close button | Save state, destroy tab |

**Visual States:**

| State | Background | Text Color | Close Button | Pin Icon |
|-------|-----------|------------|--------------|----------|
| `closed` | N/A | N/A | N/A | N/A |
| `open` | Light gray | Gray | Visible on hover | Empty |
| `active` | White | Black | Visible | Filled if pinned |
| `backgrounded` | Light gray | Gray | Visible on hover | Filled if pinned |
| `pinned` | Light gray | Black | Visible on hover | 📌 Filled |

**Tab Context Menu:**

- Close
- Close Others
- Close All
- Pin Tab
- Duplicate (for terminal/diff canvases)
- Move to New Window (detach)

---

### 2.3 Canvas State Machine

```
┌──────────────────────────────────────────────────────────────┐
│ CANVAS LIFECYCLE                                             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [empty] → [loading] → [active] ⟷ [disconnected] → [error]  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**States:**
- `empty` → No content loaded
- `loading` → Content being fetched/rendered
- `active` → Content visible and interactive
- `disconnected` → Connection lost (for iframe, terminal, etc.)
- `error` → Failed to load content

**Transitions:**

| From | To | Trigger | Action |
|------|----|---------| -------|
| `empty` | `loading` | User opens entity | Show loading spinner |
| `loading` | `active` | Content loaded | Render canvas |
| `loading` | `error` | Load failure | Show error message |
| `active` | `disconnected` | Connection lost | Show "Reconnecting..." |
| `disconnected` | `active` | Connection restored | Resume canvas |
| `active` | `empty` | Close tab | Destroy canvas instance |

**Visual States:**

| State | Display | Actions Available |
|-------|---------|-------------------|
| `empty` | Placeholder: "Select an entity to view" | None |
| `loading` | Centered spinner + "Loading {entity}..." | [Cancel] |
| `active` | Full canvas content | Context-specific actions |
| `disconnected` | Dimmed content + "Reconnecting..." banner | [Retry], [Close] |
| `error` | Error message + stack trace | [Retry], [View Logs], [Close] |

**Canvas Types & Update Frequencies:**

| Canvas Type | Update Frequency | Disconnect Behavior |
|-------------|------------------|---------------------|
| `terminal` | Real-time (stream) | Show "Connection lost", attempt reconnect |
| `iframe` | On file change (hot reload) | Show "Server offline", retry health check |
| `diff` | On demand (user navigation) | N/A (static content) |
| `chat` | Real-time (WebSocket) | Show "Reconnecting...", buffer messages |
| `heatmap` | 1s poll | Show stale timestamp, retry |
| `voice_only` | Event-driven (TTS) | Pause playback, retry on reconnect |

---

### 2.4 Project State Machine

```
┌──────────────────────────────────────────────────────────────┐
│ PROJECT LIFECYCLE                                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [inactive] → [selected] → [loading] → [ready] → [paused]   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**States:**
- `inactive` → Project exists but not selected
- `selected` → User chose project (context switching)
- `loading` → Loading project resources (daemon, dashboards, sessions)
- `ready` → All resources loaded, project active
- `paused` → Project backgrounded (resources suspended)

**Transitions:**

| From | To | Trigger | Action |
|------|----|---------| -------|
| `inactive` | `selected` | Click project in portfolio | Mark as selected |
| `selected` | `loading` | - | Load daemon state, sessions, tabs |
| `loading` | `ready` | All resources loaded | Switch UI context |
| `ready` | `paused` | Switch to another project | Pause daemons, save tabs |
| `paused` | `loading` | Switch back to project | Resume daemons, restore tabs |

**Visual States:**

| State | Badge | Opacity | Actions Available |
|-------|-------|---------|-------------------|
| `inactive` | None | 0.5 | [Select] |
| `selected` | "SELECTED" | 1.0 | [Cancel] |
| `loading` | Spinner | 1.0 | [View Logs] |
| `ready` | "ACTIVE" | 1.0 | [Pause], [View Resources] |
| `paused` | "PAUSED" | 0.7 | [Resume] |

**Project Switch Flow:**

```
User clicks [leviathan ○] (inactive)
  ↓
[leviathan ●] (selected)
  ↓
Loading daemon state...
Loading agent sessions (3)...
Loading pinned tabs (2)...
  ↓
[leviathan ●] (ready)
  ↓
UI switches context:
  - Left panel: leviathan entity tree
  - Right panel: leviathan daemon bubbles
  - Tabs: Restore 2 pinned tabs
  - Canvas: Last active tab
```

**Previous project [clawd ●] transitions to paused:**

```
Pause daemon (port 3002)
Save tab state (1 terminal, 1 diff)
Clear right panel daemon bubbles
Mark [clawd ○] as paused
```

---

### 2.5 Cognitive Layer State Machine

```
┌──────────────────────────────────────────────────────────────┐
│ COGNITIVE LAYER LIFECYCLE                                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  L0 ⟷ L1 ⟷ L2                                               │
│  (Surface) (Strategy) (Detail)                               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**States:**
- `L0_surface` → CEO view (projects, health scores, portfolio)
- `L1_strategy` → PM view (workstreams, blockers, task assignment)
- `L2_detail` → Worker view (agent sessions, terminal, diffs)

**Transitions:**

| From | To | Trigger | Action |
|------|----|---------| -------|
| `L0` | `L1` | Click project entity | Drill down into project workstreams |
| `L1` | `L2` | Click task/agent entity | Open agent session/terminal |
| `L2` | `L1` | Click breadcrumb (project level) | Zoom out to workstream view |
| `L1` | `L0` | Click breadcrumb (portfolio level) | Zoom out to portfolio view |
| `*` | `*` | Voice command "zoom in/out" | Navigate cognitive layer |

**Visual Differences:**

| Layer | Density | Entity Focus | Canvas Type | Panels |
|-------|---------|--------------|-------------|--------|
| `L0` | Minimal | Projects | Heatmap grid | Collapsed |
| `L1` | Balanced | Workstreams | Board (kanban) | Right panel visible |
| `L2` | Dense | Sessions/Tasks | Terminal/Diff | Both panels visible |

**Cognitive Layer Icons:**

- `L0`: 🌐 (Portfolio)
- `L1`: 📊 (Strategy)
- `L2`: 🔬 (Detail)

**Breadcrumb Navigation:**

```
L0: All Projects
  ↓
L0: [leviathan ●] [clawd ○] [kingly ○]
  ↓ (click leviathan)
L1: Leviathan > [core ●] [research ●] [plugins ○]
  ↓ (click core)
L2: Leviathan > Core > [daemon task]
```

**Keyboard Shortcuts:**

- `Cmd+1`: Jump to L0 (portfolio)
- `Cmd+2`: Jump to L1 (current project workstreams)
- `Cmd+3`: Jump to L2 (current entity detail)
- `Cmd+[`: Zoom out one layer
- `Cmd+]`: Zoom in one layer

---

### 2.6 Interaction Mode State Machine

```
┌──────────────────────────────────────────────────────────────┐
│ INTERACTION MODE LIFECYCLE                                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [voice_only] ⟷ [voice+visual] ⟷ [visual_only] ⟷ [keyboard_only]│
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**States:**
- `voice_only` → Jarvis mode, no visual required
- `voice+visual` → Hybrid mode (voice commands + visual feedback)
- `visual_only` → Click/mouse interaction
- `keyboard_only` → Power user mode (all shortcuts)

**Transitions:**

| From | To | Trigger | Action |
|------|----|---------| -------|
| `voice_only` | `voice+visual` | User says "show me" | Enable visual canvas |
| `voice+visual` | `visual_only` | Disable microphone | Disable voice input |
| `visual_only` | `keyboard_only` | Press `Cmd+K` (command palette) | Show shortcut hints |
| `keyboard_only` | `voice+visual` | Enable microphone | Enable voice input |
| `*` | `voice_only` | User says "voice only mode" | Collapse all panels |

**Mode-Specific UI:**

| Mode | Panels | Canvas | Shortcuts | TTS Feedback |
|------|--------|--------|-----------|--------------|
| `voice_only` | Collapsed | Minimal (text only) | Disabled | Enabled |
| `voice+visual` | Expanded | Full (terminal, iframe) | Enabled | Enabled |
| `visual_only` | Expanded | Full | Enabled | Disabled |
| `keyboard_only` | Collapsed | Full | All shortcuts visible | Disabled |

**Voice Commands Affecting Mode:**

- "Voice only mode" → `voice_only`
- "Show me the dashboard" → `voice+visual`, open iframe canvas
- "Keyboard mode" → `keyboard_only`, show shortcut overlay
- "Mouse mode" → `visual_only`, disable voice

**Visual Indicators:**

- `voice_only`: Microphone icon pulsing, minimal UI
- `voice+visual`: Microphone + eye icon
- `visual_only`: Eye icon only
- `keyboard_only`: Keyboard icon + shortcut overlay

---

## 3. Layout Space Definition

### 3.1 Dimensions

```yaml
layout_space:
  dimensions:
    cognitive_layer: [L0_surface, L1_strategy, L2_detail]
    entity_focus: [project, workstream, agent, session, daemon, task]
    interaction_mode: [voice_only, voice+visual, visual_only, keyboard_only]
    density: [minimal, balanced, dense]

  # Total theoretical permutations: 3 × 6 × 4 × 3 = 216
  # Valid permutations: ~80 (many combinations are invalid)
```

### 3.2 Valid Combinations

**Rule 1:** `voice_only + dense` is invalid (contradictory)
**Rule 2:** `L0 + session/task` is invalid (wrong abstraction level)
**Rule 3:** `keyboard_only + voice_only` is invalid (mutually exclusive)

### 3.3 Layout Presets

```yaml
presets:
  ceo_overview:
    cognitive_layer: L0_surface
    entity_focus: project
    interaction_mode: voice+visual
    density: minimal
    layout:
      canvas: heatmap_grid
      left_panel: collapsed
      right_panel: collapsed
      tabs: none

  pm_workstream:
    cognitive_layer: L1_strategy
    entity_focus: workstream
    interaction_mode: visual_only
    density: balanced
    layout:
      canvas: board_kanban
      left_panel: expanded (entity tree)
      right_panel: expanded (daemon bubbles)
      tabs: pinned workstream tabs

  worker_session:
    cognitive_layer: L2_detail
    entity_focus: session
    interaction_mode: keyboard_only
    density: dense
    layout:
      canvas: terminal
      left_panel: collapsed
      right_panel: collapsed
      tabs: all session tabs visible

  reviewer_diff:
    cognitive_layer: L2_detail
    entity_focus: agent
    interaction_mode: visual_only
    density: dense
    layout:
      canvas: diff_viewer
      left_panel: collapsed
      right_panel: expanded (PR checklist)
      tabs: all diff tabs visible

  voice_jarvis:
    cognitive_layer: L0_surface
    entity_focus: project
    interaction_mode: voice_only
    density: minimal
    layout:
      canvas: voice_only (text + TTS)
      left_panel: collapsed
      right_panel: collapsed
      tabs: none

  cowork_mode:
    cognitive_layer: L2_detail
    entity_focus: session
    interaction_mode: voice+visual
    density: balanced
    layout:
      canvas:
        left: chat_thread
        right: terminal
      left_panel: collapsed
      right_panel: expanded (agent status)
      tabs: session tabs visible

  portfolio_manager:
    cognitive_layer: L0_surface
    entity_focus: project
    interaction_mode: visual_only
    density: minimal
    layout:
      canvas: heatmap_grid
      left_panel: collapsed
      right_panel: collapsed
      tabs: none
      on_click_project: drill_down_to_L1
```

### 3.4 Layout Space Matrix

```
┌─────────────────────────────────────────────────────────────┐
│ LAYOUT SPACE MATRIX                                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│           L0              L1              L2                │
│        (Surface)      (Strategy)       (Detail)             │
│                                                             │
│ voice_only:                                                 │
│   minimal   ✓ jarvis      ✗              ✗                 │
│   balanced  ✗             ✗              ✗                 │
│   dense     ✗             ✗              ✗                 │
│                                                             │
│ voice+visual:                                               │
│   minimal   ✓ ceo         ✗              ✗                 │
│   balanced  ✗             ✓ pm           ✓ cowork          │
│   dense     ✗             ✗              ✓ reviewer        │
│                                                             │
│ visual_only:                                                │
│   minimal   ✓ portfolio   ✗              ✗                 │
│   balanced  ✗             ✓ workstream   ✗                 │
│   dense     ✗             ✗              ✓ diff            │
│                                                             │
│ keyboard_only:                                              │
│   minimal   ✗             ✗              ✗                 │
│   balanced  ✗             ✗              ✗                 │
│   dense     ✗             ✗              ✓ terminal        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.5 Dynamic Layout Adaptation

**AI suggests layout based on:**

```typescript
interface LayoutSuggestion {
  detected_user_state: {
    current_entity: 'session:abc123',
    recent_interactions: ['open terminal', 'run tests', 'view diff'],
    time_in_current_layer: '12min',
    context_switches_last_hour: 7
  },
  suggested_layout: 'worker_session', // keyboard_only + L2 + dense
  confidence: 0.89,
  rationale: 'User has been in terminal for 12min, suggesting focused work mode'
}
```

**User can:**
- Accept suggestion (apply layout)
- Modify suggestion (adjust panels/density)
- Ignore suggestion (stay in current layout)
- Save as custom preset

---

## 4. Intent Classifier Specification

### 4.1 Cognitive Layer Routing

**Voice/text input → Cognitive layer mapping**

#### L0 Queries (Portfolio/Surface)

**Pattern:** Status, overview, health, portfolio questions

```yaml
L0_patterns:
  - regex: "how are (my )?projects?( doing)?"
    cognitive_layer: L0
    entity_focus: all_projects
    canvas: heatmap_grid

  - regex: "what's the (overall )?status"
    cognitive_layer: L0
    entity_focus: all_projects
    canvas: heatmap_grid

  - regex: "any blockers?"
    cognitive_layer: L0
    entity_focus: all_projects
    canvas: heatmap_grid
    filter: blocked_only

  - regex: "show (me )?portfolio"
    cognitive_layer: L0
    entity_focus: all_projects
    canvas: heatmap_grid

  - regex: "health (check|score) for (all )?projects?"
    cognitive_layer: L0
    entity_focus: all_projects
    canvas: heatmap_grid

  - regex: "switch to ([a-z]+) project"
    cognitive_layer: L0
    entity_focus: project
    action: switch_project
    project_name: $1
```

**Examples:**
- "How are my projects?" → L0 heatmap, all projects
- "What's the status?" → L0 heatmap, all projects
- "Any blockers?" → L0 heatmap, filtered to blocked entities
- "Show me portfolio" → L0 heatmap, all projects
- "Switch to clawd project" → L0, switch to clawd project

**Confidence Threshold:** 0.80

---

#### L1 Queries (Strategy/PM)

**Pattern:** Workstream, task assignment, decision making, blocker resolution

```yaml
L1_patterns:
  - regex: "show (me )?(the )?([a-z]+) workstream"
    cognitive_layer: L1
    entity_focus: workstream
    workstream_name: $3
    canvas: board_kanban

  - regex: "what decisions need (me|approval)?"
    cognitive_layer: L1
    entity_focus: proposal
    filter: under_review
    canvas: board_kanban

  - regex: "approve (the )?plan( for ([a-z]+))?"
    cognitive_layer: L1
    entity_focus: plan
    action: approve_plan
    plan_name: $4

  - regex: "what('s|s) blocked( in ([a-z]+))?"
    cognitive_layer: L1
    entity_focus: task
    filter: blocked
    workstream_name: $3
    canvas: board_kanban

  - regex: "assign task ([a-z0-9-]+) to ([a-z]+)"
    cognitive_layer: L1
    entity_focus: task
    action: assign_task
    task_id: $1
    agent_name: $2

  - regex: "create (new )?(capability|epic|plan) for ([a-z ]+)"
    cognitive_layer: L1
    entity_focus: $2
    action: create_entity
    title: $3
```

**Examples:**
- "Show me the auth workstream" → L1 board, auth workstream
- "What decisions need me?" → L1 board, proposals under_review
- "Approve the plan" → L1, approve current plan
- "What's blocked in core?" → L1 board, core workstream, blocked tasks
- "Assign task lev-123 to worker-1" → L1, assign task
- "Create new capability for user login" → L1, create capability wizard

**Confidence Threshold:** 0.75

---

#### L2 Queries (Detail/Worker)

**Pattern:** Code, terminal, diff, logs, agent session, debugging

```yaml
L2_patterns:
  - regex: "open (the )?terminal( for ([a-z]+))?"
    cognitive_layer: L2
    entity_focus: session
    session_name: $3
    canvas: terminal

  - regex: "show (me )?(the )?diff( for ([a-z]+))?"
    cognitive_layer: L2
    entity_focus: agent
    agent_name: $4
    canvas: diff_viewer

  - regex: "run (the )?tests?( for ([a-z]+))?"
    cognitive_layer: L2
    entity_focus: session
    session_name: $3
    action: run_tests
    canvas: terminal

  - regex: "view (the )?logs?( for ([a-z]+))?"
    cognitive_layer: L2
    entity_focus: daemon
    daemon_name: $3
    canvas: terminal
    action: tail_logs

  - regex: "debug ([a-z]+)"
    cognitive_layer: L2
    entity_focus: session
    session_name: $1
    canvas: terminal
    action: start_debugger

  - regex: "what('s|s) the agent (doing|working on)?"
    cognitive_layer: L2
    entity_focus: agent
    canvas: chat_thread

  - regex: "checkout branch ([a-z0-9/-]+)"
    cognitive_layer: L2
    entity_focus: session
    action: git_checkout
    branch_name: $1
    canvas: terminal
```

**Examples:**
- "Open the terminal for daemon" → L2 terminal, daemon session
- "Show me the diff" → L2 diff_viewer, current agent
- "Run the tests for auth" → L2 terminal, run tests in auth session
- "View logs for index backend" → L2 terminal, tail index logs
- "Debug session abc123" → L2 terminal, start debugger
- "What's the agent doing?" → L2 chat_thread, show agent activity
- "Checkout branch feature/auth" → L2 terminal, git checkout

**Confidence Threshold:** 0.70

---

### 4.2 Intent Classification Algorithm

```typescript
interface IntentClassification {
  cognitive_layer: 'L0' | 'L1' | 'L2',
  entity_focus: EntityType,
  canvas: CanvasType,
  action?: string,
  filters?: Record<string, any>,
  confidence: number
}

function classifyIntent(input: string): IntentClassification {
  // Normalize input
  const normalized = input.toLowerCase().trim();

  // Try L0 patterns first (highest abstraction)
  for (const pattern of L0_patterns) {
    const match = normalized.match(pattern.regex);
    if (match && match.length > 0) {
      return {
        cognitive_layer: 'L0',
        entity_focus: pattern.entity_focus,
        canvas: pattern.canvas,
        filters: pattern.filter ? { [pattern.filter]: true } : undefined,
        confidence: calculateConfidence(match, pattern)
      };
    }
  }

  // Try L1 patterns
  for (const pattern of L1_patterns) {
    const match = normalized.match(pattern.regex);
    if (match && match.length > 0) {
      return {
        cognitive_layer: 'L1',
        entity_focus: pattern.entity_focus,
        canvas: pattern.canvas,
        action: pattern.action,
        filters: pattern.filter ? { [pattern.filter]: true } : undefined,
        confidence: calculateConfidence(match, pattern)
      };
    }
  }

  // Try L2 patterns
  for (const pattern of L2_patterns) {
    const match = normalized.match(pattern.regex);
    if (match && match.length > 0) {
      return {
        cognitive_layer: 'L2',
        entity_focus: pattern.entity_focus,
        canvas: pattern.canvas,
        action: pattern.action,
        confidence: calculateConfidence(match, pattern)
      };
    }
  }

  // Fallback: Use current cognitive layer
  return {
    cognitive_layer: getCurrentCognitiveLayer(),
    entity_focus: getCurrentEntityFocus(),
    canvas: getCurrentCanvas(),
    confidence: 0.5
  };
}

function calculateConfidence(match: RegExpMatchArray, pattern: Pattern): number {
  // Base confidence from pattern specificity
  let confidence = 0.7;

  // Increase if entity names matched
  if (pattern.entity_name && match.groups?.entity_name) {
    confidence += 0.15;
  }

  // Increase if action is explicit
  if (pattern.action) {
    confidence += 0.1;
  }

  // Cap at 0.95 (never 100% certain from pattern matching)
  return Math.min(confidence, 0.95);
}
```

### 4.3 Flowmind Integration

**Existing:** `~/lev/core/flowmind/` parses natural language → actions

**UI Integration:**

```typescript
// ~/lev/ui/adapters/flowmind-adapter.ts
interface FlowmindAdapter {
  parseIntent(input: string): IntentClassification
  executeAction(intent: IntentClassification): void
}

// Example flow:
// User: "show me blocked tasks in core workstream"
//   ↓ Flowmind parses
// Intent: {
//   cognitive_layer: L1,
//   entity_focus: 'task',
//   canvas: 'board_kanban',
//   filters: { blocked: true, workstream: 'core' },
//   confidence: 0.87
// }
//   ↓ UI executes
// - Switch to L1 (PM view)
// - Load core workstream entity tree
// - Open board_kanban canvas
// - Filter to blocked tasks only
// - Show result
```

---

## 5. Notification Routing Rules

### 5.1 Event → Cognitive Layer Routing

```yaml
notification_routing:
  L0_events:
    # Portfolio-level events (CEO needs to know)
    - project.health_degraded
    - project.all_blocked
    - initiative.milestone_reached
    - initiative.completed
    - daemon.crashed
    - dashboard.all_failed

  L1_events:
    # Strategy-level events (PM needs to know)
    - plan.blocked
    - plan.phase_completed
    - proposal.submitted
    - proposal.decision_needed
    - capability.all_tests_passing
    - workstream.all_tasks_completed
    - task.blocked
    - message.urgent

  L2_events:
    # Detail-level events (Worker needs to know)
    - session.idle_timeout
    - session.crystallization_complete
    - synth.promotion_eligible
    - artifact.ref_invalid
    - memory.decay_warning
    - agent.task_completed
    - terminal.command_failed
    - diff.comment_added
```

### 5.2 Priority/Urgency Mapping

```yaml
notification_priority:
  catastrophic:
    urgency: immediate
    visual: modal + sound alert
    cognitive_layer_override: true  # Always show, regardless of layer
    examples:
      - daemon.crashed
      - dashboard.all_failed
      - artifact.credentials_exposed

  critical:
    urgency: high
    visual: toast + badge
    cognitive_layer_override: false  # Show if in relevant layer
    examples:
      - project.all_blocked
      - plan.blocked
      - task.blocked
      - message.urgent

  warning:
    urgency: medium
    visual: badge only
    cognitive_layer_override: false
    examples:
      - session.idle_timeout
      - memory.decay_warning
      - artifact.ref_invalid

  info:
    urgency: low
    visual: silent badge increment
    cognitive_layer_override: false
    examples:
      - session.checkpoint_saved
      - task.assigned
      - message.delivered
```

### 5.3 Auto-Switch Rules

**When does UI auto-switch cognitive layers?**

```yaml
auto_switch_rules:
  catastrophic_events:
    trigger: notification.priority == 'catastrophic'
    action: switch_to_relevant_layer
    example:
      - User in L0, daemon crashes
      - Auto-switch to L2, show daemon logs

  user_preference:
    setting: "Auto-switch on critical events"
    default: false
    when_enabled:
      - catastrophic: always auto-switch
      - critical: auto-switch if idle >5min
      - warning: never auto-switch
      - info: never auto-switch

  manual_override:
    - User can disable auto-switch per session
    - Shortcut: Cmd+Shift+L (lock layer)
```

**Auto-switch example:**

```
User in L0 (portfolio view)
  ↓
Event: daemon.crashed (catastrophic)
  ↓
Auto-switch to L2 (detail view)
  ↓
Open daemon entity
  ↓
Load terminal canvas with logs
  ↓
Show modal: "Daemon crashed. View logs?"
  ↓
User clicks [View Logs]
  ↓
Display error stack trace
```

### 5.4 Badge vs Toast vs Modal

```yaml
notification_display:
  badge:
    use_for: [info, warning]
    behavior: increment_count
    location: entity_icon
    clear_on: user_views_entity

  toast:
    use_for: [warning, critical]
    behavior: auto_dismiss_after_5s
    location: bottom_right
    clear_on: auto_dismiss | user_dismisses

  modal:
    use_for: [catastrophic, critical_with_action]
    behavior: requires_user_action
    location: center_screen
    clear_on: user_takes_action

  sound:
    use_for: [catastrophic, urgent_message]
    behavior: play_alert_sound
    user_configurable: true
```

### 5.5 Notification Batching

**Problem:** Avoid notification spam (e.g., 10 tasks completed in 1s = 10 toasts)

**Solution:** Batch related events

```yaml
batching_rules:
  time_window: 2000ms  # 2 seconds

  batch_patterns:
    - event_type: task.completed
      batch_threshold: 3
      batched_message: "{count} tasks completed"

    - event_type: message.delivered
      batch_threshold: 5
      batched_message: "{count} new messages"

    - event_type: memory.decay_warning
      batch_threshold: 2
      batched_message: "{count} memories expiring soon"
```

**Example:**

```
t=0ms:   task.completed (task-1)
t=500ms: task.completed (task-2)
t=1200ms: task.completed (task-3)
  ↓
Batched notification at t=2000ms:
  Toast: "3 tasks completed"
```

---

## 6. State Diagrams (ASCII Art)

### 6.1 Full Entity Lifecycle Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ ENTITY LIFECYCLE ACROSS COGNITIVE LAYERS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ L0 (CEO):                                                       │
│   Initiative [draft] → [approved] → [in_progress] → [completed]│
│                                                                 │
│ L1 (PM):                                                        │
│   Proposal [draft] → [submitted] → [approved]                  │
│      ↓                                                          │
│   Plan [draft] → [approved] → [in_progress] → [completed]      │
│      ↓                                                          │
│   Capability [draft] → [specified] → [tested] → [released]     │
│                                                                 │
│ L2 (Worker):                                                    │
│   Session [new] → [active] → [checkpoint] → [closed]           │
│      ↓                                                          │
│   Synth [created] → [active] → [successful] → [promoted]       │
│      ↓                                                          │
│   Artifact [created] → [valid] → [accessed]                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 Project Switch Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ PROJECT CONTEXT SWITCH                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Current: [leviathan ●] (ready)                                  │
│                                                                 │
│ User clicks: [clawd ○]                                          │
│   ↓                                                             │
│ [leviathan ●] → [leviathan ○] (pausing...)                      │
│   ├─ Pause daemon (port 3001)                                   │
│   ├─ Save tab state (3 tabs)                                    │
│   ├─ Clear right panel daemon bubbles                           │
│   └─ Mark as paused                                             │
│                                                                 │
│ [clawd ○] → [clawd ●] (loading...)                              │
│   ├─ Resume daemon (port 3002)                                  │
│   ├─ Restore tab state (2 tabs)                                 │
│   ├─ Load daemon bubbles                                        │
│   └─ Mark as ready                                              │
│                                                                 │
│ UI updates:                                                     │
│   ├─ Left panel: clawd entity tree                              │
│   ├─ Right panel: clawd daemon bubbles                          │
│   ├─ Tabs: restore 2 pinned tabs                                │
│   └─ Canvas: last active tab                                    │
│                                                                 │
│ Result: [clawd ●] (ready)                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 Notification Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ NOTIFICATION ROUTING FLOW                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Event: daemon.crashed (catastrophic)                            │
│   ↓                                                             │
│ Check priority: catastrophic                                    │
│   ↓                                                             │
│ Check cognitive_layer_override: true                            │
│   ↓                                                             │
│ Current layer: L0                                               │
│   ↓                                                             │
│ Auto-switch to L2 (daemon detail)                               │
│   ├─ Navigate to daemon entity                                  │
│   ├─ Load terminal canvas with logs                             │
│   └─ Show modal: "Daemon crashed. View logs?"                   │
│                                                                 │
│ User clicks [View Logs]                                         │
│   ↓                                                             │
│ Display error stack trace in terminal                           │
│                                                                 │
│ Notification cleared                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Summary

This specification provides:

1. **Entity FSM → UI State Mapping:** All 12 entity types from `.beads/types/*.yaml` mapped to visual states with transitions, notifications, and actions
2. **Workspace State Machines:** Panel, tab, canvas, project, cognitive layer, and interaction mode FSMs
3. **Layout Space Definition:** 4D layout space with 80+ valid permutations, preset layouts, and dynamic adaptation rules
4. **Intent Classifier:** Voice/text → cognitive layer routing with pattern matching and confidence thresholds
5. **Notification Routing:** Event → cognitive layer mapping, priority levels, auto-switch rules, and batching

**Next Phase:** Stage 6 (Component Intent) will define component roles, input/output contracts, update frequencies, and interaction patterns.

---

**End of Stage 5 Specification**

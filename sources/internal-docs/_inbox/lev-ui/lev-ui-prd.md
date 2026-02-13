# Lev UI: Composable Workspace Runtime
**Product Requirements Document**

**Version:** 1.0
**Date:** 2026-02-02
**Status:** North Star
**Owner:** Leviathan Core Team

---

## Executive Summary

Lev UI is a composable workspace runtime that solves the agentic code problem: mental model collapse when running multiple concurrent AI agent projects. Unlike existing solutions (tmux, orchestration GUIs, IDE browsers, Docker), Lev UI is **project-first, not app-first**. Users assemble their own UI from primitives, the system suggests layouts based on cognitive state, and the canvas can be anything — terminal, iframe, voice, computer use — connected via adapters.

**Product Thesis:** A workspace where switching projects means everything follows. Work isn't split BETWEEN apps anymore. The UI is a function of `(user_state, entity_focus, cognitive_layer)`, not a static design.

---

## The Problem: Agentic Code Collapse

### Current State Pain Points

When developers run multiple concurrent AI agent projects:

1. **Mental model collapse** — work scattered across terminal tabs, browser windows, IDE panels with no natural grouping
2. **Context switching overhead** — more time switching than building (avg 23% productivity loss per context switch)
3. **Port collision hell** — agent A starts server on :3000, agent B tries to use :3000, manual resolution required
4. **Orphaned processes** — agents die, servers keep running, no ownership tracking
5. **No project boundary** — notification arrives, "which project was this from?" takes 10-15s to determine

### Why Existing Solutions Fail

| Solution | What It Solves | What It Doesn't |
|----------|----------------|-----------------|
| **tmux** | Terminal multiplexing | Visual context, project scoping, port management |
| **Orchestration GUIs** | Process management | Cognitive layering, customization, fractal navigation |
| **IDE Browsers** | Split views | Multi-runtime canvases, voice-first, entity-driven |
| **Docker** | Isolation | Real-time collaboration, cognitive state adaptation |
| **Background Agents** | Async execution | Visibility, session grouping, delegation tracking |

**Insight:** "We need to change how we use computers" — not just add another dashboard.

---

## Core Principles

### 1. Project-First, Not App-First

Everything is scoped to a project. Switch project = everything follows.

```yaml
# Current (app-first)
Terminal Tab 1: leviathan core work
Terminal Tab 2: clawd deployment
Terminal Tab 3: leviathan research
Browser Tab 1: localhost:3000 (which project?)
Browser Tab 2: Claude Code (which session?)

# Lev UI (project-first)
[leviathan ●]  # Active project
  ├─ 3 terminals (core, research, daemon)
  ├─ 2 canvases (localhost:3001 dashboard, diff viewer)
  ├─ 4 agent sessions (AgentPing bubbles)
  └─ Daemon: index:9200, poly-runner:8080

[clawd ○]  # Inactive, all resources paused
```

### 2. Fractal Cognitive Layering

**L0 (CEO/Surface)** → High-level status, delegation, health scores
**L1 (PM/Strategy)** → Workstreams, blockers, task assignment
**L2 (Worker/Detail)** → Agent sessions, terminal output, git diffs

Same data, different compression. Each project is an org with departments. **Zoom in and L0/L1/L2 repeats fractally.**

```
L0: All Projects    → [leviathan ●] [clawd ●] [kingly ○]
  L0: Leviathan     → [core ●] [research ●] [plugins ○]
    L0: Core        → [daemon ●] [index ●] [adapter ○]
      L1: Daemon    → 3 tasks, 1 blocked, PR pending
        L2: Task    → agent session, terminal, diff
```

**Navigation rule:** Left sidebar = entity tree. Tabs = in-flight work (not static entities). Right sidebar = contextual info (agents, daemons, workstreams).

### 3. Customizable Like Photoshop

Users build their own UI. AI suggests "here's what's possible given your cognitive state" but users compose.

**Per-cognitive-state layouts are the unique differentiator.**

```yaml
# User saves layouts per mode
layouts:
  voice_review:
    mode: voice_only
    canvas: diff_viewer
    right_panel: pr_checklist

  deep_work:
    mode: keyboard
    canvas: terminal
    left_panel: collapsed
    right_panel: collapsed

  portfolio_manager:
    mode: click
    canvas: heatmap_grid
    cognitive_layer: L0
    density: minimal
```

### 4. Agentic CMS

Users and agents can spin up new entity types, components, workflows at runtime. Everything is an entity with a lifecycle (FSM-driven).

**Existing foundation:** `.beads/types/*.yaml` (12 entity types with FSMs)

```bash
# Runtime entity creation
User: "Create a new entity type 'design-review' with states: draft, feedback, approved"
Agent: Creates .beads/types/design-review.yaml, registers FSM, generates UI components
```

### 5. shadcn Model for Components

All component code lives in your project (not node_modules). Updates are code merges.

- **For hackers/builders:** git-native, full control, fork anything
- **For non-technical users:** agentic process handles update consumption

```
~/lev/ui/components/
  ├─ agent-bubble.tsx      # Your copy, customize freely
  ├─ diff-viewer.tsx       # Your copy
  └─ terminal-canvas.tsx   # Your copy

# Updates
git remote add lev-ui-upstream https://github.com/lev-os/ui-components
git merge upstream/main --allow-unrelated-histories
# Agent: "Detected 3 component updates. Review changes?"
```

### 6. Adapter-Driven

Connect to everything. MCP, A2A, ACP, browser, terminal, FS watch.

**Existing Lev adapters to integrate:**

| Adapter | Location | Purpose |
|---------|----------|---------|
| MCP | `~/lev/core/index/src/backends/mcp.js` | External tool connections |
| AgentPing | `~/lev/core/agent-harness/vendor/AgentPing` | Agent session management |
| Polyglot Runners | `~/lev/core/polyglot-runners/registry.yaml` | Multi-runtime execution |
| Browser | TBD | Playwright/Puppeteer control |
| Terminal | TBD | PTY sessions per project |
| FS Watch | TBD | File system events → UI updates |

### 7. Scales from Voice-Only to Full IDE

Same system, different density. Voice-first user sees live results manifest. Code reviewer sees diff interface. Portfolio manager sees L0 heatmap.

---

## Architecture

### Workspace Primitives

```yaml
workspace:
  layout: user_defined | ai_suggested | preset
  cognitive_layer: [L0, L1, L2]  # affects density/abstraction
  project_context: leviathan | clawd | kingly  # everything scoped here

panels:
  - slot: left | right | top | bottom | floating
    content: <entity_type>  # agent, task, workstream, daemon
    collapsible: true
    state: expanded | collapsed | minimized

canvas:  # the main working area - can be ANYTHING
  types:
    - iframe          # localhost:3001, web dashboard
    - terminal        # shell session (PTY)
    - computer_use    # full screen agent control
    - a2ui            # agent-rendered UI (spec TBD)
    - chat            # conversation thread
    - diff            # git diff viewer
    - board           # kanban/workstream visualization
    - voice_only      # jarvis mode, no visual required
    - heatmap         # portfolio health grid (L0)

interaction_modes: [voice, click, keyboard, gesture]  # 1 or more, concurrent
```

### Layout Space (Adaptive)

The UI adapts to a **4-dimensional layout space**:

```yaml
layout_space:
  dimensions:
    cognitive_layer: [L0_surface, L1_strategy, L2_detail]
    entity_focus: [project, workstream, agent, session, task]
    interaction_mode: [voice, click, keyboard]  # can be multiple
    density: [minimal, balanced, dense]

  # Example permutations
  voice_review_mode:
    cognitive_layer: L2_detail
    entity_focus: session
    interaction_mode: voice
    density: minimal
    → Canvas: diff_viewer, Right: PR checklist, Voice: enabled

  portfolio_mode:
    cognitive_layer: L0_surface
    entity_focus: project
    interaction_mode: click
    density: minimal
    → Canvas: heatmap_grid, Panels: collapsed

  deep_work_mode:
    cognitive_layer: L2_detail
    entity_focus: task
    interaction_mode: keyboard
    density: dense
    → Canvas: terminal, Panels: collapsed, Shortcuts: enabled
```

**AI role:** Suggest layout based on `(detected_user_state, current_entity, recent_interactions)`. User can accept, modify, or ignore.

### Entity-Driven Navigation

**URI scheme:** `<project>/<entity_type>/<entity_id>` or `<entity_type>/<entity_id>` (current project implied)

```
# Examples
lev://leviathan/agent/session:abc123
lev://agent/config
lev://agent/new
lev://clawd/daemon/status
lev://workstream/frontend-refactor
```

**Navigation structure:**

- **Left sidebar:** Entity tree (projects → workstreams → agents → tasks)
- **Tabs:** In-flight work (agent sessions, terminals, diffs) — ephemeral
- **Right sidebar:** Contextual info (daemon bubbles, agent status, workstream health)
- **Everything collapsible:** Power users can collapse to just canvas

### Daemon Bubbles (Per Session)

Each agent session shows a **daemon bubble** — live indicator of processes/ports managed by that agent.

```yaml
agent_session:
  id: session:abc123
  agent: AgentPing-researcher
  daemon_bubble:
    processes:
      - name: "index backend"
        port: 9200
        status: running
        pid: 12345
      - name: "poly-runner"
        port: 8080
        status: running
        pid: 12346
    health: healthy
    last_update: 2026-02-02T14:32:01Z
```

**Visual:** Small floating bubble next to agent name in sidebar. Click to expand, see all managed processes. Agent spins up new service → appears in bubble automatically.

**Integration:** Hooks into `~/lev/core/daemon/` process management.

### Fractal Context Example

```
┌─────────────────────────────────────────────────────────────┐
│ L0: All Projects                                            │
│ ┌───────────────────────────────────────────────────────┐   │
│ │ [leviathan ●] [clawd ●] [kingly ○]                    │   │
│ │   Health: 85%    Health: 92%    Health: 100%          │   │
│ │   3 active       1 active       0 active              │   │
│ └───────────────────────────────────────────────────────┘   │
│                                                             │
│ Click [leviathan ●] →                                       │
│                                                             │
│ L0: Leviathan Project                                       │
│ ┌───────────────────────────────────────────────────────┐   │
│ │ [core ●] [research ●] [plugins ○]                     │   │
│ │   2 tasks     1 task       0 tasks                    │   │
│ └───────────────────────────────────────────────────────┘   │
│                                                             │
│ Click [core ●] →                                            │
│                                                             │
│ L0: Core Workstream                                         │
│ ┌───────────────────────────────────────────────────────┐   │
│ │ [daemon ●] [index ●] [adapter ○]                      │   │
│ └───────────────────────────────────────────────────────┘   │
│                                                             │
│ Click [daemon ●] →                                          │
│                                                             │
│ L1: Daemon Tasks (PM view)                                  │
│ ┌───────────────────────────────────────────────────────┐   │
│ │ 3 tasks in progress, 1 blocked, PR pending            │   │
│ │ Assignees: AgentPing-worker-1, AgentPing-worker-2     │   │
│ └───────────────────────────────────────────────────────┘   │
│                                                             │
│ Click task "Fix port collision" →                          │
│                                                             │
│ L2: Task Detail (Worker view)                               │
│ ┌───────────────────────────────────────────────────────┐   │
│ │ Canvas: Terminal (agent session)                      │   │
│ │ Right: Git diff, PR comments                          │   │
│ └───────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## User Personas (By Cognitive Mode)

### 1. Voice-First Visionary

**Quote:** "I don't wanna code but I wanna see a terminal and the website changing, and I wanna just talk to it and watch it manifest."

**Interaction:**
- Primary: Voice commands
- Canvas: Split screen (terminal + iframe localhost preview)
- Panels: Collapsed
- Feedback: Audio confirmations + visual progress

**Layout:**
```yaml
voice_visionary_mode:
  interaction_mode: voice
  cognitive_layer: L1_strategy
  canvas:
    left: terminal (live agent output)
    right: iframe (localhost:3001)
  panels: collapsed
  audio: enabled (TTS confirmations)
```

**Example session:**
```
User: "Create a login page with email and password fields"
Agent: [Starts work, TTS: "Creating login page"]
Canvas Left: Terminal shows agent writing code
Canvas Right: localhost:3001 hot-reloads, shows new login page
Agent: [TTS: "Login page complete. Testing form validation."]
```

### 2. Code Reviewer

**Quote:** "I need to PR review everything so I get a git diff interface and PR review workflow."

**Interaction:**
- Primary: Click + keyboard
- Canvas: Diff viewer
- Right panel: PR checklist, review comments
- Cognitive layer: L2_detail

**Layout:**
```yaml
code_reviewer_mode:
  interaction_mode: [click, keyboard]
  cognitive_layer: L2_detail
  canvas: diff_viewer
  right_panel:
    - pr_checklist
    - review_comments
    - agent_suggestions
  shortcuts:
    "a": approve change
    "c": add comment
    "r": request changes
```

### 3. Cowork Mode (Claude Cowork-style)

**Quote:** "Collaborative side-by-side with agent."

**Interaction:**
- Primary: Chat + keyboard
- Canvas: Split (chat + terminal/diff)
- Real-time collaboration

**Layout:**
```yaml
cowork_mode:
  interaction_mode: [click, keyboard]
  cognitive_layer: L2_detail
  canvas:
    left: chat_thread
    right: terminal | diff | iframe (user chooses)
  right_panel: agent_status
```

### 4. Portfolio Manager

**Quote:** "How are my 5 projects doing?"

**Interaction:**
- Primary: Click
- Canvas: Heatmap grid (all projects)
- Cognitive layer: L0_surface
- Density: Minimal

**Layout:**
```yaml
portfolio_manager_mode:
  interaction_mode: click
  cognitive_layer: L0_surface
  canvas: heatmap_grid
  density: minimal
  projects:
    - leviathan: 85% health, 3 active agents
    - clawd: 92% health, 1 active agent
    - kingly: 100% health, 0 active agents
  click_action: drill_down_to_L1
```

---

## Integration with Existing Lev Infrastructure

### Core Components to Integrate

| Lev Component | Location | UI Role |
|---------------|----------|---------|
| **Flowmind Parser** | `~/lev/core/flowmind/` | Intent → action routing (voice/text → workspace action) |
| **Daemon** | `~/lev/core/daemon/` | Process/port management per project, daemon bubble data |
| **BD (.beads)** | `~/.beads/` | Entity state + relationships, 12 types with FSMs |
| **Entity Types** | `.beads/types/*.yaml` | Data model — generate UI schema from these |
| **AgentPing** | `~/lev/core/agent-harness/vendor/AgentPing` | Agent session management, dashboard data source |
| **Polyglot Runners** | `~/lev/core/polyglot-runners/` | Multi-runtime canvas backends (Python, Node, etc.) |
| **MCP Servers** | `~/lev/core/index/src/backends/mcp.js` | External tool connections (GitHub, Slack, etc.) |
| **Index Backends** | `~/lev/core/index/src/backends/` | Search across all entities (code, docs, tasks, memory) |
| **Skills** | `~/.claude/skills/` | 171 agentic capabilities available per project |

### Entity FSM → UI State Machines

**Existing:** `.beads/types/*.yaml` defines 12 entity types with FSMs (idea, epic, task, session, etc.)

**UI Integration:**

```yaml
# .beads/types/task.yaml (existing)
task_fsm:
  states: [pending, in_progress, blocked, completed]
  transitions:
    pending → in_progress: claim_task
    in_progress → blocked: report_blocker
    blocked → in_progress: blocker_resolved
    in_progress → completed: mark_done

# UI state machine (generated)
task_ui_component:
  pending:
    color: gray
    icon: circle
    actions: [claim, assign]
  in_progress:
    color: blue
    icon: spinner
    actions: [mark_blocked, mark_done, view_session]
  blocked:
    color: red
    icon: alert
    actions: [resolve_blocker, reassign]
  completed:
    color: green
    icon: check
    actions: [reopen, archive]
```

**Code generation:** Parse `.beads/types/*.yaml` → generate React components with state-aware UI.

### Daemon Integration

**Existing:** `~/lev/core/daemon/` manages processes per project.

**UI needs:**

1. **Daemon bubble data source** — query running processes/ports
2. **Start/stop controls** — user can start/stop services from UI
3. **Health monitoring** — visual indicators (green = healthy, red = crashed)

**API contract:**

```typescript
// ~/lev/ui/adapters/daemon-adapter.ts
interface DaemonAdapter {
  listProcesses(projectId: string): Process[]
  startProcess(projectId: string, config: ProcessConfig): Process
  stopProcess(processId: string): void
  healthCheck(processId: string): HealthStatus
}

interface Process {
  id: string
  name: string
  port?: number
  pid: number
  status: 'running' | 'stopped' | 'crashed'
  startedAt: Date
  managedBy?: string  // agent session ID
}
```

### AgentPing Dashboard Data

**Existing:** `~/lev/core/agent-harness/vendor/AgentPing` has session management.

**UI integration:**

```typescript
// ~/lev/ui/adapters/agentping-adapter.ts
interface AgentPingAdapter {
  listSessions(projectId: string): AgentSession[]
  getSessionOutput(sessionId: string): SessionOutput
  sendCommand(sessionId: string, command: string): void
  terminateSession(sessionId: string): void
}

interface AgentSession {
  id: string
  agentType: string
  status: 'active' | 'idle' | 'terminated'
  startedAt: Date
  daemonBubble: DaemonBubble  // processes managed by this agent
  output: SessionOutput
}
```

### Flowmind Intent Routing

**Existing:** `~/lev/core/flowmind/` parses natural language → actions.

**UI role:** Convert voice/text commands → workspace actions.

```yaml
# Examples
intent: "Show me all blocked tasks in leviathan core"
  → cognitive_layer: L1_strategy
  → entity_focus: workstream(leviathan/core)
  → canvas: board (filtered: blocked tasks)

intent: "Switch to portfolio view"
  → cognitive_layer: L0_surface
  → entity_focus: all_projects
  → canvas: heatmap_grid

intent: "Open terminal for clawd deployment"
  → project_context: clawd
  → entity_focus: workstream(deployment)
  → canvas: terminal
  → action: spawn_pty_session
```

### Index Search Integration

**Existing:** `~/lev/core/index/src/backends/` (ast-grep, MCP, etc.)

**UI role:** Unified search across all entities (code, docs, tasks, memory, sessions).

**Component:** Global search bar (cmd+k style)

```typescript
// ~/lev/ui/components/unified-search.tsx
interface SearchResult {
  type: 'code' | 'task' | 'agent' | 'session' | 'doc' | 'memory'
  title: string
  path: string
  preview: string
  score: number
  projectId: string
}

// User types: "authentication refactor"
// Results:
// - [code] ~/lev/core/auth/refactor.ts (score: 0.95)
// - [task] lev-123: Refactor auth flow (score: 0.89)
// - [session] AgentPing session abc123 (score: 0.76)
```

---

## Component Model (shadcn-Inspired)

### Philosophy

- **Components live in YOUR project**, not in `node_modules`
- **Updates are code merges** (git-native)
- **Fork anything** — full control, customize freely
- **Agentic process** handles update consumption for non-technical users

### Component Structure

```
~/lev/ui/
  ├─ components/
  │   ├─ agent-bubble.tsx       # Agent session status bubble
  │   ├─ daemon-bubble.tsx      # Process/port status bubble
  │   ├─ diff-viewer.tsx        # Git diff canvas
  │   ├─ terminal-canvas.tsx    # PTY terminal canvas
  │   ├─ heatmap-grid.tsx       # Portfolio health grid
  │   ├─ chat-thread.tsx        # Conversation UI
  │   └─ entity-tree.tsx        # Left sidebar entity navigator
  │
  ├─ adapters/
  │   ├─ daemon-adapter.ts      # Daemon API client
  │   ├─ agentping-adapter.ts   # AgentPing API client
  │   ├─ mcp-adapter.ts         # MCP backend client
  │   └─ terminal-adapter.ts    # PTY session manager
  │
  ├─ layouts/
  │   ├─ voice-review.tsx       # Voice-first layout
  │   ├─ code-review.tsx        # Diff-focused layout
  │   ├─ cowork.tsx             # Side-by-side chat+canvas
  │   └─ portfolio.tsx          # L0 heatmap layout
  │
  └─ state/
      ├─ workspace-state.ts     # Current project, layout, cognitive layer
      ├─ entity-state.ts        # Entity tree, FSM states
      └─ session-state.ts       # Active sessions, daemon bubbles
```

### Update Flow

```bash
# For hackers/builders (git-native)
cd ~/lev/ui
git remote add upstream https://github.com/lev-os/ui-components
git fetch upstream
git merge upstream/main --allow-unrelated-histories
# Review changes, resolve conflicts, commit

# For non-technical users (agentic)
User: "Update my UI components"
Agent: [Runs git fetch/merge in background]
Agent: "Found 3 component updates: agent-bubble, diff-viewer, terminal-canvas"
Agent: "Changes preview: [shows diff]"
User: "Apply"
Agent: [Commits merge, restarts UI]
```

### Component Input/Output Contracts

```typescript
// ~/lev/ui/components/agent-bubble.tsx
interface AgentBubbleProps {
  sessionId: string
  agentType: string
  status: 'active' | 'idle' | 'terminated'
  daemonBubble: DaemonBubble
  onClick?: () => void
}

interface AgentBubbleOutput {
  event: 'expand' | 'collapse' | 'terminate'
  sessionId: string
}

// ~/lev/ui/components/terminal-canvas.tsx
interface TerminalCanvasProps {
  sessionId: string
  projectId: string
  initialCommand?: string
}

interface TerminalCanvasOutput {
  event: 'command_executed' | 'session_closed'
  sessionId: string
  exitCode?: number
}

// ~/lev/ui/components/diff-viewer.tsx
interface DiffViewerProps {
  filePath: string
  baseRef: string
  headRef: string
  projectId: string
}

interface DiffViewerOutput {
  event: 'approve' | 'comment' | 'request_changes'
  filePath: string
  comment?: string
}
```

---

## Canvas Adapter Interface

The **canvas** is the main working area and can render ANY content type via adapters.

### Canvas Types

```typescript
type CanvasType =
  | 'iframe'          // Web app, dashboard, localhost
  | 'terminal'        // Shell session (PTY)
  | 'computer_use'    // Full screen agent control
  | 'a2ui'            // Agent-rendered UI (spec TBD)
  | 'chat'            // Conversation thread
  | 'diff'            // Git diff viewer
  | 'board'           // Kanban/workstream visualization
  | 'voice_only'      // Jarvis mode, no visual
  | 'heatmap'         // Portfolio health grid (L0)
```

### Adapter Interface

```typescript
// ~/lev/ui/adapters/canvas-adapter.ts
interface CanvasAdapter {
  type: CanvasType
  render(props: CanvasProps): CanvasInstance
  updateState(instance: CanvasInstance, newState: any): void
  destroy(instance: CanvasInstance): void
}

interface CanvasProps {
  projectId: string
  entityId?: string
  config: Record<string, any>
}

interface CanvasInstance {
  id: string
  type: CanvasType
  state: Record<string, any>
  element: HTMLElement | null  // null for voice_only
}

// Example: Terminal canvas adapter
class TerminalCanvasAdapter implements CanvasAdapter {
  type = 'terminal'

  render(props: CanvasProps): CanvasInstance {
    const ptySession = spawnPtySession(props.projectId)
    const xterm = new Terminal()
    xterm.loadAddon(new WebLinksAddon())
    return {
      id: ptySession.id,
      type: 'terminal',
      state: { ptySession },
      element: xterm.element
    }
  }

  updateState(instance: CanvasInstance, command: string): void {
    instance.state.ptySession.write(command)
  }

  destroy(instance: CanvasInstance): void {
    instance.state.ptySession.kill()
  }
}
```

### Canvas Update Frequency

```yaml
canvas_update_specs:
  terminal:
    frequency: real_time  # Stream output as it arrives
    buffering: false

  iframe:
    frequency: on_change  # Hot reload on file changes
    debounce: 300ms

  diff:
    frequency: on_demand  # User navigates between files
    preload: next_file

  heatmap:
    frequency: 1s  # Poll project health every second
    batch_updates: true

  voice_only:
    frequency: on_event  # TTS when agent emits message
    queue: true
```

---

## A2UI Protocol (Agent-to-UI)

**Status:** Spec TBD (open question)

**Purpose:** Allow agents to render custom UI components dynamically.

### Proposed Contract

```typescript
// Agent emits A2UI message
interface A2UIMessage {
  type: 'a2ui'
  component: 'form' | 'chart' | 'table' | 'custom'
  props: Record<string, any>
  interactionMode: 'read_only' | 'interactive'
}

// Example: Agent renders form for user input
{
  type: 'a2ui',
  component: 'form',
  props: {
    title: 'Configure deployment',
    fields: [
      { name: 'env', type: 'select', options: ['dev', 'staging', 'prod'] },
      { name: 'region', type: 'select', options: ['us-east-1', 'eu-west-1'] }
    ]
  },
  interactionMode: 'interactive'
}

// UI renders form, user submits
// UI sends response back to agent
{
  type: 'a2ui_response',
  data: { env: 'staging', region: 'us-east-1' }
}
```

**Open questions:**

1. Security model — how to prevent malicious UI injection?
2. Component library — predefined set or arbitrary React/HTML?
3. State management — who owns form state, UI or agent?

---

## ACP Protocol (Agent Communication Protocol)

**Status:** Spec TBD (open question)

**Purpose:** Enable agents to communicate within the same project context.

### Proposed Contract

```typescript
// Agent A sends message to Agent B
interface ACPMessage {
  from: string      // Agent session ID
  to: string        // Agent session ID or 'broadcast'
  type: 'task' | 'result' | 'question' | 'notification'
  payload: any
  projectId: string
}

// Example: Research agent sends findings to implementation agent
{
  from: 'session:abc123',
  to: 'session:def456',
  type: 'result',
  payload: {
    topic: 'authentication patterns',
    findings: [...],
    confidence: 0.92
  },
  projectId: 'leviathan'
}

// UI displays notification in both agent bubbles
// Implementation agent can click to view findings
```

**Open questions:**

1. Routing — centralized message broker or peer-to-peer?
2. Persistence — store messages in BD or ephemeral?
3. Security — can agents across projects communicate?

---

## Success Criteria

### Primary Metrics

| Metric | Current (no Lev UI) | Target (with Lev UI) |
|--------|---------------------|----------------------|
| **Project notification identification time** | 10-15s | <1s |
| **Port collision incidents per week** | 3-5 | 0 |
| **Context switch time** | 8-12s | 1 click/voice command |
| **Orphaned process discovery time** | Manual, 5+ min | Real-time daemon bubbles |
| **Layout customization time** | N/A (not possible) | <2 min per layout |

### User Experience Goals

1. **Zero cognitive load for project switching** — one click, everything follows
2. **Voice-only mode is fully functional** — no visual required for visionary persona
3. **Layout saves per cognitive state** — reviewer mode, deep work mode, portfolio mode all one click away
4. **All agent sessions/terminals/outputs are project-scoped** — no more "which terminal was this?"
5. **New entity types created at runtime** — user or agent can extend the system without code changes

### Technical Goals

1. **Adapter architecture supports new canvas types** — add iframe, terminal, computer_use, etc. without core changes
2. **FSM-driven UI state** — all entity states derived from `.beads/types/*.yaml`
3. **Component updates via git merge** — shadcn model works for technical and non-technical users
4. **Sub-200ms UI response time** — layout switches, panel expansions, canvas changes feel instant
5. **Cross-platform** — works on macOS, Linux, Windows (WSL)

---

## What This Is NOT

### Market Validation ≠ Product Clone

| Product | What We Learned | What We're NOT Building |
|---------|-----------------|-------------------------|
| **8090** | Planning UX validation — fractal task decomposition works | Their specific orchestration GUI or pricing model |
| **Codex** | Clean UI primitives inspiration | Another browser-based IDE |
| **tmux** | Terminal multiplexing is insufficient | Just a better tmux |
| **Docker Desktop** | Process isolation is table stakes | Container management GUI |
| **Claude Code** | Cowork mode validates side-by-side UX | Chat-first interface |

### Core Differentiators

1. **Project-first scoping** — no other tool does this
2. **Fractal cognitive layering** — L0/L1/L2 zoom with same data
3. **Per-cognitive-state layouts** — AI-suggested, user-composed
4. **Daemon bubbles** — visual process ownership per agent
5. **Agentic CMS** — runtime entity type creation
6. **Executor abstraction** — CLI, API, webhook, anything

**The executor is abstracted** — Lev UI doesn't care HOW work gets done (CLI, API, direct code execution). It orchestrates and visualizes.

---

## Open Questions & Trade-offs

### 1. Tech Stack Decision

**Options:**

| Stack | Pros | Cons | Recommendation |
|-------|------|------|----------------|
| **Electron** | Native OS integration, mature, rich ecosystem | Heavy bundle size, memory usage | ✅ **Start here** (faster to prototype) |
| **Tauri** | Lighter weight, Rust security, smaller bundle | Less mature, smaller community | Consider for v2 optimization |
| **Web App** | No install, cross-platform by default | Limited OS integration, no PTY sessions | ❌ Doesn't meet terminal canvas requirement |

**Decision:** **Start with Electron**, migrate to Tauri post-MVP if bundle size becomes critical.

### 2. Component Framework

**Options:**

| Framework | Pros | Cons | Recommendation |
|-----------|------|------|----------------|
| **React** | Largest ecosystem, team familiarity, mature | Heavier runtime, virtual DOM overhead | ✅ **Start here** (fastest to ship) |
| **Solid** | Faster, no virtual DOM, reactive primitives | Smaller ecosystem, learning curve | Consider for performance-critical canvases |
| **Framework-agnostic** | Maximum flexibility, future-proof | Complex adapter layer, slower iteration | ❌ Over-engineering for MVP |

**Decision:** **React for MVP**, evaluate Solid for terminal/diff canvases if performance is bottleneck.

### 3. Voice Backend

**Options:**

| Backend | Pros | Cons | Recommendation |
|---------|------|------|----------------|
| **Whisper STT + Piper TTS** (from Clawd) | Local, private, no API costs | Slower than cloud, requires GPU for real-time | ✅ **Start here** (aligns with Lev philosophy) |
| **Cloud (OpenAI, Google)** | Faster, higher accuracy | API costs, privacy concerns | Fallback option for poor local performance |

**Decision:** **Whisper + Piper for MVP**, with cloud fallback toggle in settings.

**Integration path:** `~/clawd/.lev/poc/jarvis-dashboard/` already has this working.

### 4. A2UI Spec

**Open questions:**

1. **Security model** — How to prevent malicious UI injection? (sandbox iframes? CSP headers?)
2. **Component library** — Predefined set (form, chart, table) or arbitrary React/HTML?
3. **State management** — UI-owned or agent-owned?

**Proposed approach:**

- **MVP:** Predefined component set (form, chart, table, markdown)
- **Security:** Sandboxed iframe + message passing
- **State:** Agent emits schema, UI owns rendering + validation, sends final data back

**Deferred to post-MVP:** Arbitrary HTML rendering (requires security audit).

### 5. ACP Spec

**Open questions:**

1. **Routing** — Centralized message broker or peer-to-peer?
2. **Persistence** — Store messages in BD or ephemeral?
3. **Cross-project** — Can agents in different projects communicate?

**Proposed approach:**

- **MVP:** Centralized message broker (simpler, easier to debug)
- **Persistence:** Store in BD as entity type `message` (queryable, lifecycle-tracked)
- **Cross-project:** No (enforce project boundaries for security/isolation)

**Deferred to post-MVP:** Peer-to-peer for low-latency agent swarms.

---

## MVP Scope

### Stage 5: Interaction Models (Spec Phase)

**Goal:** Define FSMs for all entity types and map to UI state machines.

**Deliverables:**

1. **FSM audit** — Validate all 12 entity types in `.beads/types/*.yaml` have complete FSMs
2. **UI state machine mapping** — For each FSM state, define:
   - Visual representation (color, icon, label)
   - Available actions (buttons, shortcuts)
   - Transition rules (when can state change?)
3. **Layout space spec** — Document all permutations of `(cognitive_layer, entity_focus, interaction_mode, density)` in `layout_space.yaml`
4. **Intent classifier spec** — Define how voice/text queries map to cognitive layer changes

**Example output:**

```yaml
# ~/lev/ui/specs/layout_space.yaml
layout_space:
  cognitive_layers: [L0_surface, L1_strategy, L2_detail]
  entity_focus_types: [project, workstream, agent, session, task]
  interaction_modes: [voice, click, keyboard, gesture]
  density_levels: [minimal, balanced, dense]

  # Total permutations: 3 × 5 × 4 × 3 = 180
  # Not all valid — voice + dense = invalid, etc.

  valid_combinations:
    - cognitive_layer: L0_surface
      entity_focus: project
      interaction_mode: click
      density: minimal
      → layout: portfolio_heatmap

    - cognitive_layer: L2_detail
      entity_focus: session
      interaction_mode: voice
      density: minimal
      → layout: voice_review_mode
```

### Stage 6: Component Intent (Design Phase)

**Goal:** Define component roles, input/output contracts, update frequencies.

**Deliverables:**

1. **Component role definitions** — For each component (agent-bubble, diff-viewer, etc.):
   - Purpose (what problem does it solve?)
   - Input contract (props)
   - Output contract (events emitted)
   - Update frequency (real-time, on-demand, polling)
2. **Canvas adapter interface spec** — How do adapters register new canvas types?
3. **Panel interaction patterns** — Expand/collapse, drag-to-reorder, resize, etc.
4. **Keyboard shortcuts** — Define global shortcuts (cmd+k search, cmd+p project switch, etc.)

**Example output:**

```typescript
// ~/lev/ui/specs/component-intent.ts

// Agent Bubble Component
interface AgentBubbleIntent {
  purpose: 'Display agent session status and managed processes'

  inputs: {
    sessionId: string
    agentType: string
    status: 'active' | 'idle' | 'terminated'
    daemonBubble: DaemonBubble
  }

  outputs: {
    onExpand: () => void
    onCollapse: () => void
    onTerminate: () => void
  }

  updateFrequency: 'real_time'  // Poll daemon status every 500ms

  interactions: [
    { trigger: 'click', action: 'expand daemon bubble' },
    { trigger: 'right-click', action: 'show context menu' },
    { trigger: 'hover', action: 'show tooltip with process count' }
  ]
}

// Diff Viewer Component
interface DiffViewerIntent {
  purpose: 'Display git diff with review controls'

  inputs: {
    filePath: string
    baseRef: string
    headRef: string
    projectId: string
  }

  outputs: {
    onApprove: () => void
    onComment: (line: number, text: string) => void
    onRequestChanges: () => void
  }

  updateFrequency: 'on_demand'  // User navigates between files

  interactions: [
    { trigger: 'keyboard: a', action: 'approve change' },
    { trigger: 'keyboard: c', action: 'add comment' },
    { trigger: 'keyboard: r', action: 'request changes' },
    { trigger: 'click line number', action: 'add inline comment' }
  ]
}
```

### Post-MVP Roadmap

**Phase 1 (MVP):** Core workspace runtime

- Project-first navigation
- Entity tree (left sidebar)
- Daemon bubbles
- Terminal + iframe canvases
- Voice-only mode
- Basic layouts (voice review, code review, portfolio)

**Phase 2:** Agentic extensions

- A2UI protocol (agent-rendered forms/charts)
- ACP protocol (agent-to-agent messaging)
- Runtime entity type creation
- AI-suggested layouts based on cognitive state

**Phase 3:** Advanced canvases

- Computer use canvas (full agent control)
- Multi-agent board (kanban with agent assignments)
- Memory graph visualizer
- Real-time collaboration (multiple users, same project)

**Phase 4:** Platform

- Component marketplace (share custom components)
- Layout templates (community-contributed)
- Plugin system (custom adapters, canvases)

---

## Appendix: File Path Reference

### Existing Lev Infrastructure to Integrate

```
~/lev/core/flowmind/               # Intent parser (voice → actions)
~/lev/core/daemon/                 # Process/port management
~/lev/core/agent-harness/vendor/AgentPing  # Agent session management
~/lev/core/polyglot-runners/       # Multi-runtime execution
~/lev/core/index/src/backends/     # Search backends (ast-grep, MCP, etc.)

~/.beads/                          # BD entity storage
~/.beads/types/*.yaml              # Entity FSM definitions (12 types)

~/.claude/skills/                  # 171 agentic capabilities

~/clawd/.lev/poc/jarvis-dashboard/ # Voice backend (Whisper + Piper)
```

### New Lev UI Structure (Proposed)

```
~/lev/ui/
  ├─ src/
  │   ├─ components/               # shadcn-style components
  │   ├─ adapters/                 # Canvas, daemon, agent adapters
  │   ├─ layouts/                  # Pre-built layout templates
  │   ├─ state/                    # Workspace, entity, session state
  │   └─ main.tsx                  # Electron entry point
  │
  ├─ specs/
  │   ├─ layout_space.yaml         # Cognitive layer permutations
  │   ├─ component-intent.ts       # Component contracts
  │   ├─ a2ui-protocol.md          # Agent-to-UI spec
  │   └─ acp-protocol.md           # Agent communication spec
  │
  └─ package.json
```

---

## Summary

Lev UI is the composable workspace runtime that solves the agentic code problem. It's **project-first, fractal, customizable, and scales from voice-only to full IDE**. The UI is a function of `(user_state, entity_focus, cognitive_layer)`, not a static design. Users assemble their own workspace from primitives, agents suggest layouts based on cognitive state, and the canvas can be anything — terminal, iframe, voice, computer use — connected via adapters.

**This PRD is the north star.** All implementation work references this document. Trade-offs are documented. Open questions are explicit. Success criteria are measurable.

**Next steps:**

1. **Stage 5:** Define FSMs and layout space (spec phase)
2. **Stage 6:** Define component intent and contracts (design phase)
3. **Prototype:** Electron shell + terminal canvas + daemon bubbles
4. **Validate:** Dogfood with Lev development itself

---

**End of PRD**

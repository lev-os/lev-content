# Stage 6: Component Intent for Lev UI Workspace

**Version:** 1.0
**Date:** 2026-02-02
**Stage:** Component Intent Definition (Design Phase)
**Status:** Ready for Wireframe Design

This document defines WHAT each component in the Lev UI workspace DOES (not what it looks like), its input/output contracts, data sources, and update frequencies. This is the foundation for wireframe design.

---

## 1. Component Intent Registry

### 1.1 ProjectSelector

**Role:** Switch full workspace context to a different project

**Cognitive Layers:** L0, L1, L2 (visible at all layers)

**Input Contract:**
```typescript
interface ProjectSelectorInput {
  type: 'user_click' | 'voice_command' | 'keyboard_shortcut'
  data: {
    project_name?: string  // Direct selection
    project_id?: string    // From search/automation
  }
}
```

**Output Contract:**
```typescript
interface ProjectSelectorOutput {
  type: 'event'
  event: 'project_switched'
  data: {
    project_id: string
    project_name: string
    previous_project_id: string | null
    timestamp: string
  }
}
```

**Update Frequency:** User-driven (no polling)

**State Source:**
- `daemon-core` → active projects list
- Filesystem adapter → watches `~/.lev/projects/` and `~/.beads/`

**Data Contract:**
```typescript
interface Project {
  id: string
  name: string
  path: string
  health: number  // 0-100
  active_agents: number
  active_daemons: number
  last_activity: string
  status: 'active' | 'paused' | 'archived'
}
```

**Interactions:**
- Click project → switch context (all panels update)
- Voice: "Switch to [project]" → Flowmind → ProjectSelector
- Keyboard: `Cmd+P` → fuzzy search → select → switch

---

### 1.2 ProjectSidebar

**Role:** Display project list with status indicators and health metrics

**Cognitive Layers:** L0, L1 (collapsed in L2 for focus)

**Input Contract:**
```typescript
interface ProjectSidebarInput {
  type: 'data_update' | 'filter_change'
  data: {
    projects: Project[]
    filter?: 'all' | 'active' | 'archived'
  }
}
```

**Output Contract:**
```typescript
interface ProjectSidebarOutput {
  type: 'event'
  event: 'project_selected' | 'project_action_requested'
  data: {
    project_id: string
    action?: 'archive' | 'pause' | 'delete' | 'settings'
  }
}
```

**Update Frequency:** 1-2 second polling for health metrics + real-time daemon events

**State Source:**
- `daemon-core` → project health, active agents, process counts
- `BD` → initiative status, blocked tasks
- Filesystem watcher → new projects, deletions

**Data Contract:**
```typescript
interface ProjectSidebarState {
  projects: Array<{
    id: string
    name: string
    health: number
    indicators: {
      agents: number
      blocked_tasks: number
      daemons_running: number
      notifications: number
    }
    status: 'active' | 'paused' | 'archived'
  }>
  selected_project_id: string | null
}
```

**Interactions:**
- Click project → `ProjectSelector.switch(project_id)`
- Right-click → context menu (pause/archive/settings)
- Hover → tooltip with quick stats

---

### 1.3 TabBar

**Role:** Entity-driven tabs for in-flight work (sessions, configs, diffs)

**Cognitive Layers:** L1, L2 (primary navigation in worker modes)

**Input Contract:**
```typescript
interface TabBarInput {
  type: 'tab_open' | 'tab_close' | 'tab_focus'
  data: {
    entity_type: 'session' | 'terminal' | 'diff' | 'config' | 'board'
    entity_id: string
    canvas_type?: CanvasType
    metadata?: Record<string, any>
  }
}
```

**Output Contract:**
```typescript
interface TabBarOutput {
  type: 'event'
  event: 'tab_focused' | 'tab_closed' | 'tab_reordered'
  data: {
    entity_id: string
    entity_type: string
    canvas_type: CanvasType
  }
}
```

**Update Frequency:** Event-driven (no polling, reacts to user actions + agent events)

**State Source:**
- Session state manager → open sessions
- Canvas registry → active canvases
- User preferences → tab order, pinned tabs

**Data Contract:**
```typescript
interface Tab {
  id: string  // entity_id
  type: 'session' | 'terminal' | 'diff' | 'config' | 'board'
  title: string
  canvas_type: CanvasType
  closable: boolean
  dirty: boolean  // unsaved changes
  daemon_bubble?: DaemonBubble
  pinned: boolean
}
```

**Interactions:**
- Click tab → focus (Canvas updates)
- Close button → confirm if dirty → close
- Drag-drop → reorder tabs
- Middle-click → close tab
- Keyboard: `Cmd+W` → close, `Cmd+Shift+[/]` → prev/next tab

---

### 1.4 Canvas (Main Working Area)

**Role:** Multi-type rendering slot for primary work surface

**Cognitive Layers:** L0, L1, L2 (all layers, different canvas types per layer)

**Input Contract:**
```typescript
interface CanvasInput {
  type: 'mount' | 'update' | 'focus'
  data: {
    canvas_type: CanvasType
    entity_id: string
    config: CanvasConfig
  }
}

type CanvasType =
  | 'terminal'
  | 'iframe'
  | 'chat'
  | 'diff'
  | 'board'
  | 'voice_only'
  | 'computer_use'
  | 'a2ui'
  | 'heatmap'
```

**Output Contract:**
```typescript
interface CanvasOutput {
  type: 'event'
  event: 'canvas_ready' | 'canvas_error' | 'canvas_state_change'
  data: {
    canvas_type: CanvasType
    entity_id: string
    state: Record<string, any>
  }
}
```

**Update Frequency:** Varies by canvas type (see Canvas Adapter Interface section)

**State Source:** Adapter-specific (each canvas type has its own data source)

**Data Contract:**
```typescript
interface CanvasState {
  canvas_type: CanvasType
  entity_id: string
  adapter: CanvasAdapter
  state: Record<string, any>  // adapter-specific state
}
```

**Interactions:**
- Focus → activate canvas (keyboard/mouse events routed here)
- Canvas-specific interactions (see Canvas Adapter Interface section)

---

### 1.5 Canvas:Terminal

**Role:** PTY session with shell access, process output streaming

**Input Contract:**
```typescript
interface TerminalCanvasInput {
  session_id: string
  project_id: string
  initial_command?: string
  cwd?: string
}
```

**Output Contract:**
```typescript
interface TerminalCanvasOutput {
  event: 'command_executed' | 'session_closed' | 'process_spawned'
  session_id: string
  exit_code?: number
  pid?: number
}
```

**Update Frequency:** Real-time stream (no buffering)

**State Source:** PTY adapter → spawns pty process, streams stdout/stderr

**Adapter:** `terminal-adapter.ts` (xterm.js + node-pty)

**Data Flow:**
```
User input → Terminal component → PTY adapter → shell process
Shell process → PTY adapter → Terminal component (xterm.js) → render
```

---

### 1.6 Canvas:Iframe

**Role:** Web app preview, localhost dashboard, embedded docs

**Input Contract:**
```typescript
interface IframeCanvasInput {
  url: string
  project_id: string
  auto_reload?: boolean
  reload_pattern?: string  // glob for file changes
}
```

**Output Contract:**
```typescript
interface IframeCanvasOutput {
  event: 'iframe_loaded' | 'iframe_error' | 'reload_triggered'
  url: string
  timestamp: string
}
```

**Update Frequency:** On-demand + hot-reload (300ms debounce on file changes)

**State Source:**
- URL provided by user/agent
- FS watcher → triggers reload on file changes

**Adapter:** `iframe-adapter.ts` (postMessage bridge for 2-way communication)

**Data Flow:**
```
FS watcher → detects change → debounce → iframe.reload()
Iframe content → postMessage → adapter → UI events
```

---

### 1.7 Canvas:Chat

**Role:** Conversation thread with agent session

**Input Contract:**
```typescript
interface ChatCanvasInput {
  session_id: string
  agent_type: string
  project_id: string
  message_history?: Message[]
}
```

**Output Contract:**
```typescript
interface ChatCanvasOutput {
  event: 'message_sent' | 'message_received' | 'typing_indicator'
  session_id: string
  message?: Message
}
```

**Update Frequency:** Real-time WebSocket stream

**State Source:**
- `agent-harness` → WebSocket connection to agent session
- BD → message persistence

**Adapter:** `agentping-adapter.ts` (WebSocket client)

**Data Flow:**
```
User message → Chat component → WebSocket → agent session
Agent session → WebSocket → Chat component → render + persist to BD
```

---

### 1.8 Canvas:Diff

**Role:** Git diff viewer with review controls

**Input Contract:**
```typescript
interface DiffCanvasInput {
  file_path: string
  base_ref: string
  head_ref: string
  project_id: string
}
```

**Output Contract:**
```typescript
interface DiffCanvasOutput {
  event: 'approve' | 'comment' | 'request_changes'
  file_path: string
  comment?: { line: number, text: string }
}
```

**Update Frequency:** On-demand (user navigates between files)

**State Source:** Git adapter → reads diffs from .git/

**Adapter:** `git-adapter.ts` (diff parsing + syntax highlighting)

**Data Flow:**
```
User selects file → Git adapter → parse diff → render with syntax highlighting
User adds comment → persist to BD → sync with git notes
```

---

### 1.9 Canvas:Board

**Role:** Kanban-style workstream visualization from BD issues

**Input Contract:**
```typescript
interface BoardCanvasInput {
  workstream_id?: string
  project_id: string
  filters?: {
    assignee?: string
    priority?: string
    labels?: string[]
  }
}
```

**Output Contract:**
```typescript
interface BoardCanvasOutput {
  event: 'task_moved' | 'task_created' | 'task_clicked'
  task_id: string
  new_state?: string
}
```

**Update Frequency:** 1 second polling + real-time BD events

**State Source:**
- BD → tasks, epics, initiatives
- `.beads/types/*.yaml` → FSM states for columns

**Adapter:** `bd-adapter.ts` (BD query API)

**Data Flow:**
```
BD query → parse tasks by state → render columns
User drags task → update BD state → trigger FSM transition
BD event → update UI in real-time
```

---

### 1.10 Canvas:VoiceOnly

**Role:** Jarvis mode - no visual, audio-only interaction

**Input Contract:**
```typescript
interface VoiceOnlyCanvasInput {
  project_id: string
  session_id?: string
  tts_enabled: boolean
  stt_enabled: boolean
}
```

**Output Contract:**
```typescript
interface VoiceOnlyCanvasOutput {
  event: 'voice_command' | 'tts_complete' | 'stt_error'
  command?: string
  response?: string
}
```

**Update Frequency:** Real-time audio stream

**State Source:**
- Whisper STT → voice input
- Piper TTS → voice output
- Flowmind → intent parsing

**Adapter:** `voice-adapter.ts` (integrates with ~/clawd/.lev/poc/jarvis-dashboard/)

**Data Flow:**
```
User speech → Whisper STT → text command → Flowmind parser → workspace action
Agent response → Piper TTS → audio output → user hears result
```

---

### 1.11 Canvas:ComputerUse

**Role:** Full-screen agent control (screenshot + input relay)

**Input Contract:**
```typescript
interface ComputerUseCanvasInput {
  session_id: string
  agent_type: string
  target_window?: string
}
```

**Output Contract:**
```typescript
interface ComputerUseCanvasOutput {
  event: 'screenshot_taken' | 'input_sent' | 'control_released'
  session_id: string
  screenshot?: string  // base64
}
```

**Update Frequency:** Real-time stream (agent-driven)

**State Source:**
- Agent session → requests screenshots, sends input events
- OS APIs → capture screen, send keyboard/mouse events

**Adapter:** `computer-use-adapter.ts` (Playwright/Puppeteer-based)

**Data Flow:**
```
Agent request → screenshot API → capture → send to agent
Agent action → input relay → OS event API → execute
```

---

### 1.12 Canvas:A2UI

**Role:** Agent-rendered UI components (forms, charts, tables)

**Input Contract:**
```typescript
interface A2UICanvasInput {
  session_id: string
  component_spec: {
    type: 'form' | 'chart' | 'table' | 'custom'
    props: Record<string, any>
    interaction_mode: 'read_only' | 'interactive'
  }
}
```

**Output Contract:**
```typescript
interface A2UICanvasOutput {
  event: 'a2ui_response' | 'a2ui_error'
  session_id: string
  data: Record<string, any>
}
```

**Update Frequency:** Event-driven (agent sends updates, user submits)

**State Source:**
- Agent session → emits A2UI message
- UI component → renders based on spec

**Adapter:** `a2ui-adapter.ts` (sandboxed iframe or React component registry)

**Data Flow:**
```
Agent → A2UI message → component spec → render sandboxed component
User interaction → component → emit response → send to agent
```

**Security:** Sandboxed iframe with CSP headers, predefined component set only (MVP)

---

### 1.13 Canvas:Heatmap

**Role:** Portfolio health grid for L0 (CEO) view

**Input Contract:**
```typescript
interface HeatmapCanvasInput {
  projects: Project[]
  metric: 'health' | 'velocity' | 'blocked_tasks'
}
```

**Output Contract:**
```typescript
interface HeatmapCanvasOutput {
  event: 'project_selected' | 'drill_down'
  project_id: string
}
```

**Update Frequency:** 1 second polling (project health metrics)

**State Source:**
- `daemon-core` → project health
- BD → initiative status, blocked tasks

**Adapter:** `heatmap-adapter.ts` (D3.js or custom rendering)

**Data Flow:**
```
Daemon poll → aggregate project metrics → render heatmap cells
User click cell → drill down to L1 (project detail view)
```

---

### 1.14 AgentList

**Role:** Display all agents in current project with status

**Cognitive Layers:** L1, L2 (summary in L1, full detail in L2)

**Input Contract:**
```typescript
interface AgentListInput {
  type: 'data_update' | 'filter_change'
  data: {
    project_id: string
    agents: AgentSession[]
    filter?: 'all' | 'active' | 'idle' | 'terminated'
  }
}
```

**Output Contract:**
```typescript
interface AgentListOutput {
  type: 'event'
  event: 'agent_selected' | 'agent_action_requested'
  data: {
    session_id: string
    action?: 'focus' | 'terminate' | 'restart' | 'view_logs'
  }
}
```

**Update Frequency:** 500ms polling + real-time AgentPing events

**State Source:**
- `AgentPing` → agent session states
- BD → task assignments
- Daemon → managed processes per agent

**Data Contract:**
```typescript
interface AgentSession {
  id: string
  agent_type: string
  status: 'active' | 'idle' | 'terminated'
  started_at: string
  task_id?: string
  daemon_bubble: DaemonBubble
  output_preview: string  // last 3 lines
}
```

**Interactions:**
- Click agent → open chat canvas (or existing session tab)
- Right-click → context menu (terminate/restart/view full logs)
- Hover → tooltip with daemon bubble processes

---

### 1.15 DaemonPanel

**Role:** Per-project daemon status (all subsystems, ports, health)

**Cognitive Layers:** L1, L2 (collapsed in L0)

**Input Contract:**
```typescript
interface DaemonPanelInput {
  type: 'data_update'
  data: {
    project_id: string
    subsystems: SubsystemStatus[]
  }
}
```

**Output Contract:**
```typescript
interface DaemonPanelOutput {
  type: 'event'
  event: 'subsystem_action_requested'
  data: {
    subsystem_name: string
    action: 'start' | 'stop' | 'restart' | 'view_logs'
  }
}
```

**Update Frequency:** 1 second polling

**State Source:** `daemon-core` CLI commands
- `lev daemon status` → overall status
- `lev daemon subsystems` → per-subsystem detail
- `lev daemon ports` → port assignments

**Data Contract:**
```typescript
interface SubsystemStatus {
  name: 'index' | 'queue' | 'watcher' | 'poly' | 'flowmind'
  status: 'running' | 'stopped' | 'crashed' | 'starting'
  port?: number
  pid?: number
  restarts: number
  health?: 'healthy' | 'degraded' | 'unhealthy'
  last_heartbeat?: string
}
```

**Interactions:**
- Click subsystem → expand detail (logs, metrics)
- Start/Stop buttons → call `lev daemon subsystems start/stop [name]`
- Port click → open in browser (if HTTP service)

---

### 1.16 DaemonBubble

**Role:** Per-session daemon indicator (processes managed by agent)

**Cognitive Layers:** L2 (visible on agent session tabs)

**Input Contract:**
```typescript
interface DaemonBubbleInput {
  type: 'data_update'
  data: {
    session_id: string
    daemon_bubble: DaemonBubble
  }
}

interface DaemonBubble {
  processes: Array<{
    name: string
    port?: number
    pid: number
    status: 'running' | 'stopped' | 'crashed'
  }>
  health: 'healthy' | 'degraded' | 'unhealthy'
  last_update: string
}
```

**Output Contract:**
```typescript
interface DaemonBubbleOutput {
  type: 'event'
  event: 'bubble_clicked' | 'process_action_requested'
  data: {
    session_id: string
    process_name?: string
    action?: 'stop' | 'restart' | 'view_logs'
  }
}
```

**Update Frequency:** 500ms polling (only for active sessions)

**State Source:**
- `daemon-core` → processes owned by session
- Agent metadata → which processes this session spawned

**Data Flow:**
```
Agent spawns process → daemon tracks ownership → bubble shows process
User clicks bubble → expand → see all processes + health
User clicks process → action menu (stop/restart/logs)
```

**Interactions:**
- Click bubble → expand/collapse process list
- Hover → tooltip with quick stats (2 processes, 1 port)
- Click process → context menu

---

### 1.17 WorkstreamBoard

**Role:** Initiative/task progress visualization per workstream

**Cognitive Layers:** L0 (summary), L1 (detail)

**Input Contract:**
```typescript
interface WorkstreamBoardInput {
  type: 'data_update' | 'filter_change'
  data: {
    workstreams: Workstream[]
    view_mode: 'summary' | 'detail'
  }
}

interface Workstream {
  id: string
  name: string
  initiative_id?: string
  tasks: Task[]
  blocked_count: number
  completed_count: number
  total_count: number
}
```

**Output Contract:**
```typescript
interface WorkstreamBoardOutput {
  type: 'event'
  event: 'workstream_selected' | 'task_action_requested'
  data: {
    workstream_id: string
    task_id?: string
    action?: 'assign' | 'block' | 'complete' | 'view_detail'
  }
}
```

**Update Frequency:** 1 second polling + real-time BD events

**State Source:**
- BD → initiatives, tasks, FSM states
- `.beads/types/*.yaml` → entity schemas, state machines

**Data Contract:**
```typescript
interface Task {
  id: string
  title: string
  state: string  // FSM state from .beads/types/
  priority: string
  assignee?: string
  blocked_by?: string[]
}
```

**Interactions:**
- L0 view: Single metric per workstream (% complete, blocked count)
- L1 view: Full kanban board with columns per FSM state
- Drag-drop task → update BD state → FSM transition
- Click task → open detail view (canvas switches to task detail)

---

### 1.18 NotificationToast

**Role:** Project-scoped alerts and status messages

**Cognitive Layers:** L0, L1, L2 (all layers)

**Input Contract:**
```typescript
interface NotificationToastInput {
  type: 'notification'
  data: {
    project_id: string
    severity: 'info' | 'warning' | 'error' | 'success'
    title: string
    message: string
    source: 'daemon' | 'agent' | 'bd' | 'system'
    action?: {
      label: string
      callback: () => void
    }
  }
}
```

**Output Contract:**
```typescript
interface NotificationToastOutput {
  type: 'event'
  event: 'notification_dismissed' | 'notification_action_clicked'
  data: {
    notification_id: string
  }
}
```

**Update Frequency:** Event-driven (no polling)

**State Source:**
- `daemon-core` → process crashes, port collisions
- `AgentPing` → agent errors, task completions
- BD → entity state transitions (task blocked, proposal approved)

**Data Flow:**
```
Event source → notification queue → toast renderer (3 second auto-dismiss)
User click action → execute callback → dismiss toast
User dismiss → remove from queue
```

**Interactions:**
- Auto-dismiss after 3 seconds (unless severity = error)
- Click action button → execute callback
- Click X → manual dismiss
- Queue: Max 3 toasts visible, others queued

---

### 1.19 SpawnWizard

**Role:** Interactive wizard to create new agent/session/daemon

**Cognitive Layers:** L1, L2

**Input Contract:**
```typescript
interface SpawnWizardInput {
  type: 'open_wizard'
  data: {
    entity_type: 'agent' | 'session' | 'daemon' | 'task'
    project_id: string
    prefill?: Record<string, any>
  }
}
```

**Output Contract:**
```typescript
interface SpawnWizardOutput {
  type: 'event'
  event: 'entity_created' | 'wizard_cancelled'
  data: {
    entity_id?: string
    entity_type: string
  }
}
```

**Update Frequency:** User-driven (modal/wizard flow)

**State Source:**
- `.beads/types/*.yaml` → entity schemas, creation prompts
- User input → form values

**Data Flow:**
```
User opens wizard → load entity schema → render form (1 question at a time)
User answers → validate → next question
Final step → create entity in BD → spawn agent/daemon if needed
```

**Interactions:**
- Multi-step wizard (matches `.beads/types/*/creation_prompt`)
- Back/Next navigation
- Cancel → discard, Submit → create entity
- Voice mode: Ask questions verbally, accept voice answers

---

### 1.20 CognitiveLayerSwitcher

**Role:** Toggle between L0/L1/L2 views (or auto-detect)

**Cognitive Layers:** Meta (visible at all layers)

**Input Contract:**
```typescript
interface CognitiveLayerSwitcherInput {
  type: 'layer_change' | 'auto_detect'
  data: {
    layer?: 'L0' | 'L1' | 'L2'
    auto_detect?: boolean
  }
}
```

**Output Contract:**
```typescript
interface CognitiveLayerSwitcherOutput {
  type: 'event'
  event: 'layer_changed'
  data: {
    previous_layer: 'L0' | 'L1' | 'L2'
    new_layer: 'L0' | 'L1' | 'L2'
    auto_detected: boolean
  }
}
```

**Update Frequency:** User-driven or auto (based on cognitive state detection)

**State Source:**
- User preference
- Cognitive state detector (analyzes current entity focus + interaction patterns)

**Data Flow:**
```
User clicks L0/L1/L2 → workspace re-renders (component visibility + density changes)
Auto mode: Monitor user activity → detect pattern → suggest layer switch
```

**Interactions:**
- Click L0/L1/L2 button → instant switch
- Toggle auto mode → enable/disable cognitive state detection
- Keyboard: `Cmd+0/1/2` → switch to L0/L1/L2

---

### 1.21 InteractionModeSelector

**Role:** Toggle voice/click/keyboard interaction modes

**Cognitive Layers:** Meta (visible at all layers)

**Input Contract:**
```typescript
interface InteractionModeSelectorInput {
  type: 'mode_toggle'
  data: {
    modes: Array<'voice' | 'click' | 'keyboard'>
    exclusive?: boolean  // true = only one mode, false = multiple concurrent
  }
}
```

**Output Contract:**
```typescript
interface InteractionModeSelectorOutput {
  type: 'event'
  event: 'interaction_modes_changed'
  data: {
    active_modes: Array<'voice' | 'click' | 'keyboard'>
  }
}
```

**Update Frequency:** User-driven

**State Source:** User preference + voice adapter availability

**Data Flow:**
```
User toggles voice → enable/disable voice adapter
User toggles keyboard → enable/disable keyboard shortcuts
Click always enabled (no toggle)
```

**Interactions:**
- Toggle switches for each mode
- Voice requires STT/TTS adapters available
- Multiple modes can be active simultaneously

---

### 1.22 LayoutEditor

**Role:** Photoshop-like workspace customizer (save layouts per cognitive state)

**Cognitive Layers:** Meta (accessed via settings)

**Input Contract:**
```typescript
interface LayoutEditorInput {
  type: 'open_editor'
  data: {
    current_layout: LayoutConfig
    presets: LayoutConfig[]
  }
}

interface LayoutConfig {
  name: string
  cognitive_layer: 'L0' | 'L1' | 'L2'
  panels: {
    left?: PanelConfig
    right?: PanelConfig
    top?: PanelConfig
    bottom?: PanelConfig
  }
  canvas: CanvasConfig
  interaction_modes: Array<'voice' | 'click' | 'keyboard'>
  density: 'minimal' | 'balanced' | 'dense'
}
```

**Output Contract:**
```typescript
interface LayoutEditorOutput {
  type: 'event'
  event: 'layout_saved' | 'layout_loaded' | 'layout_deleted'
  data: {
    layout_id: string
    layout: LayoutConfig
  }
}
```

**Update Frequency:** User-driven

**State Source:**
- User preferences → saved layouts
- Preset library → default layouts

**Data Flow:**
```
User opens editor → load current layout → drag-drop panels, resize, configure
User saves → persist to ~/.lev/ui/layouts/{name}.yaml
User loads preset → apply layout → re-render workspace
```

**Interactions:**
- Drag-drop panels to reposition
- Resize panel widths/heights
- Add/remove panels
- Save as preset
- Load preset

---

### 1.23 SettingsModal

**Role:** Project/agent/workspace configuration

**Cognitive Layers:** Meta (accessed via settings)

**Input Contract:**
```typescript
interface SettingsModalInput {
  type: 'open_settings'
  data: {
    scope: 'workspace' | 'project' | 'agent'
    entity_id?: string
  }
}
```

**Output Contract:**
```typescript
interface SettingsModalOutput {
  type: 'event'
  event: 'settings_updated' | 'settings_cancelled'
  data: {
    scope: string
    changes: Record<string, any>
  }
}
```

**Update Frequency:** User-driven

**State Source:**
- `.lev/config.yaml` → project settings
- `~/.lev/ui/config.yaml` → workspace settings
- BD → agent configurations

**Data Flow:**
```
User opens settings → load config → render form
User changes values → validate → save to config file
Reload adapters/components if needed
```

**Interactions:**
- Tabbed interface (Workspace / Project / Agent)
- Form fields with validation
- Apply → save changes
- Cancel → discard changes

---

### 1.24 SearchOmnibar

**Role:** Unified search across all entities (lev find integration)

**Cognitive Layers:** L0, L1, L2 (all layers)

**Input Contract:**
```typescript
interface SearchOmnibarInput {
  type: 'search_query'
  data: {
    query: string
    filters?: {
      type?: Array<'code' | 'task' | 'agent' | 'session' | 'doc' | 'memory'>
      project_id?: string
    }
  }
}
```

**Output Contract:**
```typescript
interface SearchOmnibarOutput {
  type: 'event'
  event: 'search_result_selected' | 'search_cancelled'
  data: {
    result?: SearchResult
  }
}

interface SearchResult {
  type: 'code' | 'task' | 'agent' | 'session' | 'doc' | 'memory'
  title: string
  path: string
  preview: string
  score: number
  project_id: string
}
```

**Update Frequency:** Debounced input (300ms) + instant results

**State Source:** `lev find` (unified search backend)
- `core/index/src/backends/backend-registry.js` → all search backends
- Backends: `ck`, `leann`, `qmd`, `bd`, `automem`, `ideas`, `sessions`, `ast-grep`

**Data Flow:**
```
User types query → debounce 300ms → call lev find → aggregate results
Render results grouped by type (code/tasks/docs/memory)
User selects result → navigate to entity (open in canvas)
```

**Interactions:**
- Keyboard: `Cmd+K` → open omnibar
- Type query → instant search
- Arrow keys → navigate results
- Enter → select result
- Esc → close omnibar

---

## 2. Canvas Adapter Interface

The Canvas is the main working area and can render ANY content type via adapters.

### 2.1 Core Adapter Interface

```typescript
/**
 * Canvas Adapter Interface
 * All canvas types implement this contract
 */
interface CanvasAdapter {
  /** Canvas type identifier */
  type: CanvasType

  /**
   * Mount the canvas into a DOM container
   * @param container - HTML element to mount into
   * @param config - Canvas-specific configuration
   * @returns Canvas instance with state
   */
  mount(container: HTMLElement, config: CanvasConfig): CanvasInstance

  /**
   * Unmount and cleanup the canvas
   * @param instance - Canvas instance to destroy
   */
  unmount(instance: CanvasInstance): void

  /**
   * Send a message/command to the canvas
   * @param instance - Canvas instance
   * @param message - Adapter-specific message
   */
  send(instance: CanvasInstance, message: AdapterMessage): void

  /**
   * Subscribe to canvas events
   * @param instance - Canvas instance
   * @param event - Event name
   * @param handler - Event handler
   * @returns Unsubscribe function
   */
  on(instance: CanvasInstance, event: string, handler: EventHandler): () => void

  /**
   * Get current canvas state
   * @param instance - Canvas instance
   * @returns Canvas state snapshot
   */
  getState(instance: CanvasInstance): CanvasState
}

/** Canvas type enum */
type CanvasType =
  | 'terminal'
  | 'iframe'
  | 'chat'
  | 'diff'
  | 'board'
  | 'voice_only'
  | 'computer_use'
  | 'a2ui'
  | 'heatmap'

/** Canvas configuration (adapter-specific) */
interface CanvasConfig {
  entity_id: string
  project_id: string
  [key: string]: any  // Adapter-specific config
}

/** Canvas instance with state */
interface CanvasInstance {
  id: string
  type: CanvasType
  state: Record<string, any>
  element: HTMLElement | null  // null for voice_only
}

/** Adapter message (adapter-specific) */
interface AdapterMessage {
  type: string
  data: any
}

/** Event handler */
type EventHandler = (event: any) => void

/** Canvas state snapshot */
interface CanvasState {
  [key: string]: any
}
```

---

### 2.2 Terminal Adapter Implementation

**File:** `~/lev/ui/adapters/terminal-adapter.ts`

```typescript
import { Terminal } from 'xterm'
import { FitAddon } from 'xterm-addon-fit'
import { WebLinksAddon } from 'xterm-addon-web-links'
import { spawn as spawnPty } from 'node-pty'

export class TerminalAdapter implements CanvasAdapter {
  type = 'terminal' as const

  mount(container: HTMLElement, config: CanvasConfig): CanvasInstance {
    const { entity_id, project_id, cwd, initialCommand } = config

    // Spawn PTY process
    const ptyProcess = spawnPty('zsh', [], {
      name: 'xterm-color',
      cwd: cwd || process.env.HOME,
      env: {
        ...process.env,
        LEV_PROJECT: project_id,
        LEV_SESSION: entity_id,
      },
    })

    // Create xterm.js terminal
    const xterm = new Terminal({
      cursorBlink: true,
      fontSize: 14,
      fontFamily: 'Menlo, Monaco, monospace',
    })

    const fitAddon = new FitAddon()
    xterm.loadAddon(fitAddon)
    xterm.loadAddon(new WebLinksAddon())

    xterm.open(container)
    fitAddon.fit()

    // Pipe PTY output to xterm
    ptyProcess.on('data', (data) => {
      xterm.write(data)
    })

    // Pipe xterm input to PTY
    xterm.onData((data) => {
      ptyProcess.write(data)
    })

    // Execute initial command if provided
    if (initialCommand) {
      ptyProcess.write(`${initialCommand}\n`)
    }

    return {
      id: entity_id,
      type: 'terminal',
      state: { ptyProcess, xterm, fitAddon },
      element: container,
    }
  }

  unmount(instance: CanvasInstance): void {
    const { ptyProcess, xterm } = instance.state
    ptyProcess.kill()
    xterm.dispose()
  }

  send(instance: CanvasInstance, message: AdapterMessage): void {
    const { ptyProcess } = instance.state
    if (message.type === 'command') {
      ptyProcess.write(`${message.data}\n`)
    }
  }

  on(instance: CanvasInstance, event: string, handler: EventHandler): () => void {
    const { ptyProcess } = instance.state
    if (event === 'exit') {
      ptyProcess.on('exit', handler)
      return () => ptyProcess.off('exit', handler)
    }
    return () => {}
  }

  getState(instance: CanvasInstance): CanvasState {
    const { ptyProcess } = instance.state
    return {
      pid: ptyProcess.pid,
      running: !ptyProcess.killed,
    }
  }
}
```

**Update Frequency:** Real-time stream (no buffering)

**Data Source:** PTY process (node-pty)

---

### 2.3 Iframe Adapter Implementation

**File:** `~/lev/ui/adapters/iframe-adapter.ts`

```typescript
import { watch } from 'chokidar'
import { debounce } from 'lodash'

export class IframeAdapter implements CanvasAdapter {
  type = 'iframe' as const

  mount(container: HTMLElement, config: CanvasConfig): CanvasInstance {
    const { entity_id, url, autoReload, reloadPattern } = config

    const iframe = document.createElement('iframe')
    iframe.src = url
    iframe.style.width = '100%'
    iframe.style.height = '100%'
    iframe.style.border = 'none'
    container.appendChild(iframe)

    let watcher: any = null
    if (autoReload && reloadPattern) {
      // Watch filesystem for changes
      watcher = watch(reloadPattern, { ignoreInitial: true })
      const reload = debounce(() => {
        iframe.contentWindow?.location.reload()
      }, 300)
      watcher.on('change', reload)
    }

    return {
      id: entity_id,
      type: 'iframe',
      state: { iframe, watcher, url },
      element: iframe,
    }
  }

  unmount(instance: CanvasInstance): void {
    const { iframe, watcher } = instance.state
    if (watcher) watcher.close()
    iframe.remove()
  }

  send(instance: CanvasInstance, message: AdapterMessage): void {
    const { iframe } = instance.state
    if (message.type === 'postMessage') {
      iframe.contentWindow?.postMessage(message.data, '*')
    } else if (message.type === 'reload') {
      iframe.contentWindow?.location.reload()
    }
  }

  on(instance: CanvasInstance, event: string, handler: EventHandler): () => void {
    if (event === 'message') {
      const listener = (e: MessageEvent) => {
        if (e.source === instance.state.iframe.contentWindow) {
          handler(e.data)
        }
      }
      window.addEventListener('message', listener)
      return () => window.removeEventListener('message', listener)
    }
    return () => {}
  }

  getState(instance: CanvasInstance): CanvasState {
    return {
      url: instance.state.url,
      loaded: instance.state.iframe.contentDocument?.readyState === 'complete',
    }
  }
}
```

**Update Frequency:** On-demand + hot-reload (300ms debounce)

**Data Source:** URL (localhost or external)

---

### 2.4 Chat Adapter Implementation

**File:** `~/lev/ui/adapters/agentping-adapter.ts`

```typescript
import { io, Socket } from 'socket.io-client'

export class ChatAdapter implements CanvasAdapter {
  type = 'chat' as const

  mount(container: HTMLElement, config: CanvasConfig): CanvasInstance {
    const { entity_id, project_id, agentType } = config

    // Connect to AgentPing WebSocket
    const socket = io(`ws://localhost:${config.agentPingPort || 8080}`, {
      query: {
        sessionId: entity_id,
        projectId: project_id,
      },
    })

    const messageHistory: any[] = []

    socket.on('message', (message) => {
      messageHistory.push(message)
      // Trigger UI update
    })

    return {
      id: entity_id,
      type: 'chat',
      state: { socket, messageHistory },
      element: container,
    }
  }

  unmount(instance: CanvasInstance): void {
    const { socket } = instance.state
    socket.disconnect()
  }

  send(instance: CanvasInstance, message: AdapterMessage): void {
    const { socket } = instance.state
    if (message.type === 'sendMessage') {
      socket.emit('message', message.data)
    }
  }

  on(instance: CanvasInstance, event: string, handler: EventHandler): () => void {
    const { socket } = instance.state
    socket.on(event, handler)
    return () => socket.off(event, handler)
  }

  getState(instance: CanvasInstance): CanvasState {
    return {
      messageCount: instance.state.messageHistory.length,
      connected: instance.state.socket.connected,
    }
  }
}
```

**Update Frequency:** Real-time WebSocket stream

**Data Source:** AgentPing session (WebSocket)

---

### 2.5 Diff Adapter Implementation

**File:** `~/lev/ui/adapters/git-adapter.ts`

```typescript
import { execSync } from 'child_process'
import { parse as parseDiff } from 'diff'

export class DiffAdapter implements CanvasAdapter {
  type = 'diff' as const

  mount(container: HTMLElement, config: CanvasConfig): CanvasInstance {
    const { entity_id, filePath, baseRef, headRef, projectId } = config

    // Execute git diff
    const diffOutput = execSync(
      `git diff ${baseRef}..${headRef} -- ${filePath}`,
      { cwd: projectId, encoding: 'utf-8' }
    )

    const parsedDiff = parseDiff(diffOutput)

    return {
      id: entity_id,
      type: 'diff',
      state: { filePath, baseRef, headRef, parsedDiff },
      element: container,
    }
  }

  unmount(instance: CanvasInstance): void {
    // No cleanup needed
  }

  send(instance: CanvasInstance, message: AdapterMessage): void {
    // Diff is read-only, no send implementation
  }

  on(instance: CanvasInstance, event: string, handler: EventHandler): () => void {
    // Events emitted by UI component (approve, comment, etc.)
    return () => {}
  }

  getState(instance: CanvasInstance): CanvasState {
    return {
      filePath: instance.state.filePath,
      baseRef: instance.state.baseRef,
      headRef: instance.state.headRef,
      lineCount: instance.state.parsedDiff[0]?.hunks?.length || 0,
    }
  }
}
```

**Update Frequency:** On-demand (user navigates between files)

**Data Source:** Git repository

---

### 2.6 Board Adapter Implementation

**File:** `~/lev/ui/adapters/bd-adapter.ts`

```typescript
import { execSync } from 'child_process'

export class BoardAdapter implements CanvasAdapter {
  type = 'board' as const

  mount(container: HTMLElement, config: CanvasConfig): CanvasInstance {
    const { entity_id, project_id, workstreamId, filters } = config

    // Query BD for tasks
    const query = workstreamId
      ? `bd list --workstream ${workstreamId} --format json`
      : `bd list --project ${project_id} --format json`

    const tasks = JSON.parse(execSync(query, { encoding: 'utf-8' }))

    // Poll for updates every 1 second
    const intervalId = setInterval(() => {
      const updatedTasks = JSON.parse(execSync(query, { encoding: 'utf-8' }))
      // Trigger UI update
    }, 1000)

    return {
      id: entity_id,
      type: 'board',
      state: { tasks, intervalId },
      element: container,
    }
  }

  unmount(instance: CanvasInstance): void {
    const { intervalId } = instance.state
    clearInterval(intervalId)
  }

  send(instance: CanvasInstance, message: AdapterMessage): void {
    if (message.type === 'updateTask') {
      const { taskId, newState } = message.data
      execSync(`bd update ${taskId} --state ${newState}`)
      // Trigger immediate poll
    }
  }

  on(instance: CanvasInstance, event: string, handler: EventHandler): () => void {
    // Events emitted by UI component (task_moved, task_clicked, etc.)
    return () => {}
  }

  getState(instance: CanvasInstance): CanvasState {
    return {
      taskCount: instance.state.tasks.length,
    }
  }
}
```

**Update Frequency:** 1 second polling + real-time BD events (future)

**Data Source:** BD (beads issue tracker)

---

### 2.7 Voice-Only Adapter Implementation

**File:** `~/lev/ui/adapters/voice-adapter.ts`

```typescript
import { spawn } from 'child_process'

export class VoiceAdapter implements CanvasAdapter {
  type = 'voice_only' as const

  mount(container: HTMLElement, config: CanvasConfig): CanvasInstance {
    const { entity_id, project_id, ttsEnabled, sttEnabled } = config

    let sttProcess: any = null
    let ttsProcess: any = null

    if (sttEnabled) {
      // Spawn Whisper STT process
      sttProcess = spawn('whisper-cli', ['--continuous'])
      sttProcess.stdout.on('data', (data: Buffer) => {
        const text = data.toString().trim()
        // Emit voice_command event with text
      })
    }

    if (ttsEnabled) {
      // Spawn Piper TTS process
      ttsProcess = spawn('piper-cli', ['--model', 'en_US-lessac-medium'])
    }

    return {
      id: entity_id,
      type: 'voice_only',
      state: { sttProcess, ttsProcess },
      element: null,  // No visual element
    }
  }

  unmount(instance: CanvasInstance): void {
    const { sttProcess, ttsProcess } = instance.state
    if (sttProcess) sttProcess.kill()
    if (ttsProcess) ttsProcess.kill()
  }

  send(instance: CanvasInstance, message: AdapterMessage): void {
    const { ttsProcess } = instance.state
    if (message.type === 'speak' && ttsProcess) {
      ttsProcess.stdin.write(`${message.data}\n`)
    }
  }

  on(instance: CanvasInstance, event: string, handler: EventHandler): () => void {
    // Events: voice_command, tts_complete, stt_error
    return () => {}
  }

  getState(instance: CanvasInstance): CanvasState {
    return {
      sttRunning: !!instance.state.sttProcess,
      ttsRunning: !!instance.state.ttsProcess,
    }
  }
}
```

**Update Frequency:** Real-time audio stream

**Data Source:** Whisper STT + Piper TTS (from ~/clawd/.lev/poc/jarvis-dashboard/)

---

### 2.8 Computer Use Adapter Implementation

**File:** `~/lev/ui/adapters/computer-use-adapter.ts`

```typescript
import { chromium } from 'playwright'

export class ComputerUseAdapter implements CanvasAdapter {
  type = 'computer_use' as const

  async mount(container: HTMLElement, config: CanvasConfig): Promise<CanvasInstance> {
    const { entity_id, sessionId } = config

    // Launch browser in agent control mode
    const browser = await chromium.launch({ headless: false })
    const page = await browser.newPage()

    return {
      id: entity_id,
      type: 'computer_use',
      state: { browser, page },
      element: container,
    }
  }

  async unmount(instance: CanvasInstance): Promise<void> {
    const { browser } = instance.state
    await browser.close()
  }

  async send(instance: CanvasInstance, message: AdapterMessage): Promise<void> {
    const { page } = instance.state
    if (message.type === 'screenshot') {
      const screenshot = await page.screenshot({ encoding: 'base64' })
      // Emit screenshot event with data
    } else if (message.type === 'click') {
      await page.click(message.data.selector)
    } else if (message.type === 'type') {
      await page.type(message.data.selector, message.data.text)
    }
  }

  on(instance: CanvasInstance, event: string, handler: EventHandler): () => void {
    // Events: screenshot_taken, input_sent, control_released
    return () => {}
  }

  getState(instance: CanvasInstance): CanvasState {
    return {
      browserRunning: !!instance.state.browser,
    }
  }
}
```

**Update Frequency:** Real-time (agent-driven)

**Data Source:** Playwright/Puppeteer OS API

---

### 2.9 A2UI Adapter Implementation

**File:** `~/lev/ui/adapters/a2ui-adapter.ts`

```typescript
export class A2UIAdapter implements CanvasAdapter {
  type = 'a2ui' as const

  mount(container: HTMLElement, config: CanvasConfig): CanvasInstance {
    const { entity_id, componentSpec } = config

    // Create sandboxed iframe
    const iframe = document.createElement('iframe')
    iframe.sandbox.add('allow-scripts', 'allow-forms')
    iframe.style.width = '100%'
    iframe.style.height = '100%'
    iframe.style.border = 'none'

    // Inject component based on spec
    const componentHTML = this.renderComponent(componentSpec)
    iframe.srcdoc = componentHTML

    container.appendChild(iframe)

    return {
      id: entity_id,
      type: 'a2ui',
      state: { iframe, componentSpec },
      element: iframe,
    }
  }

  unmount(instance: CanvasInstance): void {
    const { iframe } = instance.state
    iframe.remove()
  }

  send(instance: CanvasInstance, message: AdapterMessage): void {
    const { iframe } = instance.state
    if (message.type === 'updateProps') {
      iframe.contentWindow?.postMessage(
        { type: 'updateProps', data: message.data },
        '*'
      )
    }
  }

  on(instance: CanvasInstance, event: string, handler: EventHandler): () => void {
    if (event === 'a2ui_response') {
      const listener = (e: MessageEvent) => {
        if (e.source === instance.state.iframe.contentWindow) {
          handler(e.data)
        }
      }
      window.addEventListener('message', listener)
      return () => window.removeEventListener('message', listener)
    }
    return () => {}
  }

  getState(instance: CanvasInstance): CanvasState {
    return {
      componentType: instance.state.componentSpec.type,
    }
  }

  private renderComponent(spec: any): string {
    // Predefined component set: form, chart, table
    if (spec.type === 'form') {
      return this.renderForm(spec.props)
    } else if (spec.type === 'chart') {
      return this.renderChart(spec.props)
    } else if (spec.type === 'table') {
      return this.renderTable(spec.props)
    }
    throw new Error(`Unknown component type: ${spec.type}`)
  }

  private renderForm(props: any): string {
    // Generate HTML form from props.fields
    return `<form>...</form>`
  }

  private renderChart(props: any): string {
    // Generate Chart.js HTML
    return `<canvas id="chart">...</canvas>`
  }

  private renderTable(props: any): string {
    // Generate HTML table
    return `<table>...</table>`
  }
}
```

**Update Frequency:** Event-driven (agent sends updates, user submits)

**Data Source:** Agent session (A2UI protocol message)

**Security:** Sandboxed iframe with CSP headers

---

### 2.10 Heatmap Adapter Implementation

**File:** `~/lev/ui/adapters/heatmap-adapter.ts`

```typescript
import { execSync } from 'child_process'

export class HeatmapAdapter implements CanvasAdapter {
  type = 'heatmap' as const

  mount(container: HTMLElement, config: CanvasConfig): CanvasInstance {
    const { entity_id, projects, metric } = config

    // Poll daemon for project metrics every 1 second
    const intervalId = setInterval(() => {
      const metrics = projects.map((project: any) => {
        const status = JSON.parse(
          execSync(`lev daemon status --project ${project.id} --format json`, {
            encoding: 'utf-8',
          })
        )
        return {
          project_id: project.id,
          health: status.health,
          blocked_tasks: status.blocked_tasks,
          velocity: status.velocity,
        }
      })
      // Trigger UI update with metrics
    }, 1000)

    return {
      id: entity_id,
      type: 'heatmap',
      state: { intervalId },
      element: container,
    }
  }

  unmount(instance: CanvasInstance): void {
    const { intervalId } = instance.state
    clearInterval(intervalId)
  }

  send(instance: CanvasInstance, message: AdapterMessage): void {
    // Heatmap is read-only, no send implementation
  }

  on(instance: CanvasInstance, event: string, handler: EventHandler): () => void {
    // Events emitted by UI component (project_selected, drill_down, etc.)
    return () => {}
  }

  getState(instance: CanvasInstance): CanvasState {
    return {}
  }
}
```

**Update Frequency:** 1 second polling

**Data Source:** `lev daemon status` (project health metrics)

---

## 3. Data Flow Contracts

### 3.1 Component → Data Source Mapping

```yaml
# Component → Data Source
ProjectSelector:
  - daemon-core (active projects)
  - filesystem (watches ~/.lev/projects/)

ProjectSidebar:
  - daemon-core (health, agents, processes)
  - BD (initiative status, blocked tasks)
  - filesystem (new/deleted projects)

TabBar:
  - session state manager (open sessions)
  - canvas registry (active canvases)
  - user preferences (tab order, pinned)

Canvas:Terminal:
  - PTY adapter (node-pty)

Canvas:Iframe:
  - URL (localhost or external)
  - filesystem watcher (hot-reload)

Canvas:Chat:
  - AgentPing WebSocket (agent session)
  - BD (message persistence)

Canvas:Diff:
  - Git repository (.git/)

Canvas:Board:
  - BD (tasks, epics, initiatives)
  - .beads/types/*.yaml (FSM states)

Canvas:VoiceOnly:
  - Whisper STT (voice input)
  - Piper TTS (voice output)
  - Flowmind (intent parsing)

Canvas:ComputerUse:
  - Playwright/Puppeteer (OS API)

Canvas:A2UI:
  - Agent session (A2UI protocol message)

Canvas:Heatmap:
  - daemon-core (project health)
  - BD (initiative status)

AgentList:
  - AgentPing (agent session states)
  - BD (task assignments)
  - daemon-core (managed processes)

DaemonPanel:
  - daemon-core (subsystem status)
  - lev daemon CLI (ports, health)

DaemonBubble:
  - daemon-core (processes owned by session)

WorkstreamBoard:
  - BD (initiatives, tasks, FSM states)
  - .beads/types/*.yaml (entity schemas)

NotificationToast:
  - daemon-core (process crashes, port collisions)
  - AgentPing (agent errors, task completions)
  - BD (entity state transitions)

SpawnWizard:
  - .beads/types/*.yaml (entity schemas, creation prompts)
  - BD (entity creation)

CognitiveLayerSwitcher:
  - user preferences
  - cognitive state detector (activity analysis)

InteractionModeSelector:
  - user preferences
  - voice adapter availability

LayoutEditor:
  - user preferences (saved layouts)
  - preset library (default layouts)

SettingsModal:
  - .lev/config.yaml (project settings)
  - ~/.lev/ui/config.yaml (workspace settings)
  - BD (agent configurations)

SearchOmnibar:
  - lev find (unified search backend)
  - All backends: ck, leann, qmd, bd, automem, ideas, sessions, ast-grep
```

---

## 4. Component Composition Rules

### 4.1 L0 (CEO/Surface Layer)

**Purpose:** Portfolio management, high-level delegation

**Visible Components:**
- ProjectSelector (prominent, always visible)
- WorkstreamBoard (summary mode: % complete, blocked count per workstream)
- NotificationToast (critical alerts only)
- Canvas: `heatmap` OR `voice_only` OR `board` (summary view)

**Hidden/Collapsed:**
- TabBar (no in-flight work at L0)
- AgentList (agents are abstracted away)
- DaemonPanel (infrastructure details hidden)
- DaemonBubble (not relevant)

**Density:** Minimal (large fonts, high-level metrics)

**Interaction Modes:** Click OR voice (keyboard less common)

**Layout Example:**
```
┌────────────────────────────────────────────────────────────┐
│ [LEV UI]  ProjectSelector: [leviathan ●] [clawd ●] [...]  │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Canvas: Heatmap Grid                                      │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  [leviathan]  85%  ●●●○○                             │  │
│  │  [clawd]      92%  ●●●●○                             │  │
│  │  [kingly]    100%  ●●●●●                             │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  Workstream Summary:                                       │
│  • Core: 3 tasks, 1 blocked                                │
│  • Research: 2 tasks, 0 blocked                            │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

### 4.2 L1 (PM/Strategy Layer)

**Purpose:** Workstream management, task assignment, blocker resolution

**Visible Components:**
- ProjectSelector (visible but smaller)
- TabBar (workstreams, not sessions)
- AgentList (summary: agent count, status)
- WorkstreamBoard (detail mode: full kanban with FSM columns)
- DaemonPanel (collapsed, expandable)
- NotificationToast (all project alerts)
- Canvas: `board` OR `chat` OR `iframe` (dashboard)

**Hidden/Collapsed:**
- DaemonBubble (not needed at workstream level)

**Density:** Balanced (readable, moderate detail)

**Interaction Modes:** Click + keyboard (common shortcuts)

**Layout Example:**
```
┌────────────────────────────────────────────────────────────┐
│ [LEV UI]  [leviathan ●] [clawd ○]  │ TabBar: [core] [research] │
├─────────────────────────────────┬──────────────────────────┤
│ WorkstreamBoard (core)          │ AgentList:               │
│ ┌─────────┬─────────┬─────────┐ │ • AgentPing-1 (active)   │
│ │ Pending │ In Prog │ Done    │ │ • AgentPing-2 (idle)     │
│ ├─────────┼─────────┼─────────┤ │                          │
│ │ Task A  │ Task B  │ Task C  │ │ DaemonPanel (collapsed)  │
│ │ Task D  │         │         │ │ ▸ 4 subsystems running   │
│ └─────────┴─────────┴─────────┘ │                          │
│                                 │                          │
└─────────────────────────────────┴──────────────────────────┘
```

---

### 4.3 L2 (Worker/Detail Layer)

**Purpose:** Deep work, agent session focus, terminal output

**Visible Components:**
- TabBar (sessions, prominent - primary navigation)
- AgentList (full detail: output preview, daemon bubbles)
- DaemonPanel (expanded with subsystem details)
- DaemonBubble (per-session indicator)
- NotificationToast (all alerts)
- Canvas: `terminal` OR `diff` OR `chat` OR `computer_use`

**Hidden/Collapsed:**
- ProjectSelector (collapsed to save space, expandable)
- WorkstreamBoard (not needed in detail view)

**Density:** Dense (small fonts, max information)

**Interaction Modes:** Keyboard (heavy shortcut use) + click

**Layout Example:**
```
┌────────────────────────────────────────────────────────────┐
│ TabBar: [session-abc ●] [terminal-xyz] [diff-main...feat]  │
├─────────────────────────────────┬──────────────────────────┤
│ Canvas: Terminal (session-abc)  │ AgentList:               │
│ ┌─────────────────────────────┐ │ • session-abc (active)   │
│ │ $ npm run dev               │ │   ○ index:9200           │
│ │ > Server started on :3001   │ │   ○ poly:8080            │
│ │ $ git status                │ │   [last 3 lines...]      │
│ │ On branch feat/ui           │ │                          │
│ │ ...                         │ │ DaemonPanel:             │
│ └─────────────────────────────┘ │ • index (running, :9200) │
│                                 │ • queue (running)        │
│                                 │ • watcher (running)      │
└─────────────────────────────────┴──────────────────────────┘
```

---

## 5. shadcn-Style Component Distribution

### 5.1 Component Registry

**Concept:** Components live in YOUR project, not in `node_modules`. Updates are git merges.

**Registry Structure:**
```
~/lev/ui/
  ├─ components/
  │   ├─ agent-bubble.tsx       # Your copy
  │   ├─ daemon-bubble.tsx      # Your copy
  │   ├─ diff-viewer.tsx        # Your copy
  │   ├─ terminal-canvas.tsx    # Your copy
  │   ├─ heatmap-grid.tsx       # Your copy
  │   ├─ chat-thread.tsx        # Your copy
  │   └─ entity-tree.tsx        # Your copy
  │
  ├─ adapters/
  │   ├─ terminal-adapter.ts    # Your copy
  │   ├─ iframe-adapter.ts      # Your copy
  │   ├─ agentping-adapter.ts   # Your copy
  │   ├─ bd-adapter.ts          # Your copy
  │   └─ voice-adapter.ts       # Your copy
  │
  └─ layouts/
      ├─ voice-review.tsx       # Your copy
      ├─ code-review.tsx        # Your copy
      └─ portfolio.tsx          # Your copy
```

---

### 5.2 Component Installation (shadcn-style)

**Command:** `npx lev-ui add <component>`

**Example:**
```bash
# Install agent-bubble component
npx lev-ui add agent-bubble

# Output:
✓ Copied agent-bubble.tsx to ~/lev/ui/components/agent-bubble.tsx
✓ Copied agent-bubble.types.ts to ~/lev/ui/components/agent-bubble.types.ts
✓ Updated ~/lev/ui/components/index.ts
```

**Behind the scenes:**
1. Fetch component source from `https://github.com/lev-os/ui-components`
2. Copy to your project
3. Update import index

---

### 5.3 Component Update Flow (Git-Native)

**For hackers/builders:**

```bash
cd ~/lev/ui
git remote add upstream https://github.com/lev-os/ui-components
git fetch upstream
git merge upstream/main --allow-unrelated-histories

# Review changes
git diff HEAD~1 components/agent-bubble.tsx

# Resolve conflicts if needed
# Commit merge
```

**For non-technical users (agentic):**

```bash
User: "Update my UI components"

Agent: [Runs git fetch/merge in background]
Agent: "Found 3 component updates:"
       • agent-bubble.tsx (new daemon indicator)
       • diff-viewer.tsx (syntax highlighting fix)
       • terminal-canvas.tsx (performance optimization)

Agent: "Review changes?"

User: "Yes"

Agent: [Shows diffs using Canvas:Diff]

User: "Apply"

Agent: [Commits merge, restarts UI if needed]
Agent: "✓ Components updated. UI restarted."
```

---

### 5.4 Custom Component Creation

**User creates new component:**

```bash
# Create new component
touch ~/lev/ui/components/my-custom-widget.tsx

# Write component code (follows same interface contracts)
# Register in ~/lev/ui/components/index.ts

# Use in layout
import { MyCustomWidget } from './components/my-custom-widget'
```

**Component must follow interface contracts:**
- Input/output contracts (see Component Intent Registry)
- Update frequency declaration
- State source declaration
- Cognitive layer visibility rules

---

### 5.5 Component Update Changelog (Agentic Process)

**Changelog format:** `CHANGELOG.md` in upstream repo

```markdown
# Changelog

## 2026-02-02

### agent-bubble.tsx
- Added real-time daemon indicator
- Fixed hover tooltip positioning
- BREAKING: Changed `DaemonBubble` interface (added `health` field)

### diff-viewer.tsx
- Improved syntax highlighting performance
- Added line number click-to-comment

## 2026-01-15
...
```

**Agentic merge process:**

1. Agent reads `CHANGELOG.md`
2. Agent analyzes breaking changes
3. Agent proposes merge strategy:
   - Non-breaking: Auto-merge
   - Breaking: Show diff, ask user approval
4. User approves
5. Agent applies merge, updates dependent code if needed

---

## 6. Next Steps: Wireframe Design

This document provides the foundation for wireframe design. The next stage (Stage 7) should produce:

1. **Low-fidelity wireframes** for each cognitive layer (L0, L1, L2)
2. **Component interaction flows** (click paths, keyboard shortcuts, voice commands)
3. **Responsive layout specs** (window sizes, panel resize behavior)
4. **Visual design language** (colors, typography, iconography)

**Key wireframe deliverables:**

- L0 Portfolio View (heatmap + project selector)
- L1 Workstream View (board + agent list + tabs)
- L2 Deep Work View (terminal + diff + daemon panel)
- Canvas type transitions (how canvas changes when switching entities)
- Panel expand/collapse animations
- Notification toast positioning and queueing
- Search omnibar interaction flow

---

**End of Stage 6 Component Intent Specification**

# Lev Features Not Represented in Documentation

> **Generated**: 2026-02-13
> **Sources**: core/, plugins/, context/, workshop/, .lev/pm/, .beads/ — cross-referenced with docs/
> **Purpose**: Audit of implemented features that lack or have minimal documentation

---

## Executive Summary

The documentation (`docs/01-architecture.md`, `docs/design/*`, `docs/specs/*`) covers roughly half of the implemented surface. Many core modules, plugins, and patterns exist in code and conversations but are absent or underdocumented.

---

## 1. CORE Modules — Missing or Underdocumented

| Module | Description | Doc Status |
|--------|-------------|------------|
| **core/exec** | FlowMind-gated agent execution SDK. Polyglot adapter routing, gate pipeline, trace/store. **THE** universal execution entry point for CLI, MCP, SDK, poly. | **NOT DOCUMENTED** — Critical runtime piece; 01-architecture mentions harness/flowmind but not exec SDK |
| **core/legos** | Lego Builder — schema-validated context assembly, generation, composition. `assembleContexts()`, `generateContext()`, `composeContexts()`, templates, validation pipeline. | **NOT DOCUMENTED** |
| **core/mesh** | Distributed mesh networking — mothership/satellite registration. | Mentioned as "planned" in spec-kernel; **no design doc** |
| **core/auth-sniffer** | Browser cookie authentication discovery with fallback chain. | Listed in 01-architecture map only; **no design doc** |
| **core/platform-detection** | Detects AI-assisted coding platforms (Claude Code, Cursor, Aider, Windsurf, Continue, Cline). | Listed only; **no doc** |
| **core/channels** | Multi-channel notification delivery with OpenClaw integration. | Listed only; **no doc** |
| **core/cdo** | Cognitive Diversity Orchestration — complexity router, skill discovery, bd_orchestrator coroutine. | **NOT DOCUMENTED** — Only beads/handoffs reference it |
| **core/fractal** | Convention-based module scanner for fractal architecture. | Vernacular mentions `lib/fractal`→`core/fractal`; **no module design** |
| **core/adapters** | Generic adapter layer. | **Unclear role** — may overlap exec/adapters |
| **core/agent-harness** | Separate from harness; vendor/AgentPing, component migration. | **NOT DOCUMENTED** |
| **core/skills-registry** | Skills registry and discovery. | Mentioned in plugins; **no core doc** |
| **core/lib** | Shared utilities: config/, fractal/, middleware/, watch/. No package.json. | **NOT DOCUMENTED** — lev-02w6 tracked packaging |
| **core/validation** | Validators: mathematical, consensus, opposition, parliament, breakthrough bubbler. | **NOT DOCUMENTED** — lev-08of tracks validator→agent gap |
| **core/commands** | Command registry/dispatch. | Implicit in CLI design; **no standalone doc** |
| **core/contexts** | Universal context library. | **NOT DOCUMENTED** — Legos references it |
| **core/debug** | LevLogger, debug utilities. | **NOT DOCUMENTED** |

---

## 2. PLUGINS — Missing or Underdocumented

| Plugin | Description | Doc Status |
|--------|-------------|------------|
| **@lev-os/codex** | LLM-first programming knowledge crystallization. Paradigms, frameworks, languages, semantic search, hexagonal architecture. | **docs/plugins/cms.md** mentions; **no codex doc** |
| **@lev-os/strands-tools** | Comprehensive Python tool integration and workflow automation. | **NOT DOCUMENTED** |
| **@lev-os/gemini-executor** | Gemini-powered workflow executor with XState (Mastra-inspired). | Mentioned in core-other bypass list; **no doc** |
| **@lev-os/workflow-orchestrator** | Bi-directional workflow orchestration — LLMs orchestrate themselves. | **NOT DOCUMENTED** |
| **@lev-os/jared-intelligence** | Jared AI intelligence coordinator — Slack/Notion integration, fractal adapter. | **NOT DOCUMENTED** |
| **@lev-os/eeps-system** | Eight Essential Psychological States — 8-personality integration. | **NOT DOCUMENTED** |
| **@lev-os/constitutional-ai** | Constitutional AI with neurochemical optimization. | **NOT DOCUMENTED** |
| **@lev-os/constitutional-framework** | Optional constitutional framework for responsible AI. | **NOT DOCUMENTED** |
| **@lev-os/vision** | Vision processing and web automation. | **NOT DOCUMENTED** |
| **@lev-os/wiggum-marketer** | Content empire marketing automation. | **NOT DOCUMENTED** |
| **@lev-os/writer** | Writer plugin with projects, contexts, workflows (whitepaper, eeps, cdo, dmt). | **NOT DOCUMENTED** |
| **@lev-os/devshop** | — | **NOT DOCUMENTED** |
| **@lev-os/dotfiles-dashboard** | — | **NOT DOCUMENTED** |
| **@lev-os/trending** | — | **NOT DOCUMENTED** |
| **@lev-os/tmux-orch** | Tmux orchestration. | **NOT DOCUMENTED** |
| **plugins/@homie** | — | **NOT DOCUMENTED** |

**01-architecture mentions**: workshop, timetravel, automem, graphiti, agentfs, openwork, skill-seekers, deploy, platforms, sdlc, scheduling — most have partial spec coverage.

---

## 3. CONTEXT System — Missing Docs

| Path | Content | Doc Status |
|------|---------|------------|
| **context/** | Patterns (systematic-opposition, figure-storming, agile-scrum), lev-self proposals, schemas. | **NOT DOCUMENTED** |
| **context/.archive** | Entity types (workspace, folder, project, epic, task, portfolio, ticket, emergent), agent templates (EEPS, strategic, analytical, creative), personality modes (NFJ-Visionary, STP-Adapter, etc.), research prompts. | **NOT DOCUMENTED** |
| **context/schemas/** | stream-router.yaml, flowmind-types.yaml (referenced in specs). | **NOT DOCUMENTED** |
| **os/contexts** | — | **NOT DOCUMENTED** |

---

## 4. WORKSHOP — Missing Docs

| Path | Content | Doc Status |
|------|---------|------------|
| **workshop/** | Top-level workshop directory. | **Cursorignore blocks**; assumed POC/skill dev area |
| **plugins/workshop** | ships-with plugin. | **Partial** — architecture mentions CDO, team coordination |
| **plugins/workshop/contexts** | — | **NOT DOCUMENTED** |
| **.ck/workshop** | — | **NOT DOCUMENTED** |

---

## 5. Features from .lev/pm and Beads — Not in Docs

| Feature | Source | Doc Status |
|---------|--------|------------|
| **FlowMind chat/voice reconciliation** | spec-flowmind-chat-voice-reconciliation-2026-02-13, proposal 2026-02-13 | Spec in .lev/pm; **not promoted to docs/specs** |
| **Deterministic orchestration validation** | .lev/pm/validation-reports/ | **NOT IN DOCS** |
| **domain.verb CLI** (`lev idea.capture`, `lev task.complete`) | lev-071f.29 | **NOT IN DOCS** |
| **Protocol handlers** (bd://, file://, prompt://, review-queue://) | lev-071f.31 | **NOT IN DOCS** |
| **Agent memory spaces in Navigator** | lev-071f.32 | **NOT IN DOCS** |
| **Auto-emit BD templates from FlowMind** | lev-071f.30 | **NOT IN DOCS** |
| **Overlay system** (Phase 6 — Kustomize-style merges, CUE validation, fork routing) | lev-082d | **NOT IN DOCS** |
| **Cognee poly integration** (Neo4j, 9 retrievers, LSMTree) | lev-0kyo | **NOT IN DOCS** |
| **Lifecycle status line + Ollama classification** | lev-0n4a, spec-lifecycle-statusline-ollama-classifier | Spec may exist; **not in docs/specs** |
| **5-tier memory upgrade** | proposal 2026-02-10 | **NOT IN DOCS** |
| **UX for web app** (wireframes, task graph, IA schema) | .lev/ux/20260212-* | **NOT IN DOCS** |
| **Trace/store, exec traces, snapshots** | .lev/exec/traces/, core/harness trace | **NOT IN DOCS** |
| **Validation gates** (UBS scan, lev-code-review) | .lev/validators/ | **NOT IN DOCS** |

---

## 6. APPS — Missing Docs

| App | Description | Doc Status |
|-----|-------------|------------|
| **apps/max** | — | **NOT DOCUMENTED** |
| **apps/shift-app** | BMAD teams, ice-breaker, web bundles. | **NOT DOCUMENTED** |
| **apps/video** | — | **NOT DOCUMENTED** |
| **apps/landing-astro** | — | **NOT DOCUMENTED** |
| **apps/auth-proxy** | — | **NOT DOCUMENTED** |
| **apps/universal-preview** | — | **NOT DOCUMENTED** |
| **apps/old-dash** | — | **NOT DOCUMENTED** |

01-architecture lists apps/desktop, expo, nextjs, flutter — others not covered.

---

## 7. CRATES — Partial Coverage

| Crate | Doc Status |
|-------|------------|
| **lev-agentfs** | spec-agentfs, crate docs |
| **lev-reactive** | Listed in README |
| **lev-tui-*** | Listed in README |
| **lev-desktop** | Listed |
| **lev-ui-universal** | Listed |

Crate-local docs exist per AGENTS.md; **no system-wide feature index** for Rust surface.

---

## 8. Community Packages — Not in Main Docs

| Package | Doc Status |
|---------|------------|
| **agentping** | spec-agentping, spec-agentping-remote, spec-agentping-surfaces |
| **lev-portable** | — |
| **agent-lease** | Architecture mentions |
| **kingly-assistant** | — |
| **clawcast** | — |
| **lev-content** | — |

---

## 9. Consolidated Feature List (One Huge List)

Below is the flattened list of all features that are **implemented or actively planned** but **not adequately represented** in docs.

### Core
- exec (FlowMind-gated execution SDK, adapters, gate, trace)
- legos (assembly, generation, composition, templates, validation)
- mesh (mothership/satellite, distributed registration)
- auth-sniffer
- platform-detection
- channels (OpenClaw, multi-channel notifications)
- cdo (Cognitive Diversity Orchestration, bd_orchestrator)
- fractal (module scanner)
- agent-harness (AgentPing vendor, component migration)
- skills-registry
- lib (config, fractal, middleware, watch)
- validation (mathematical, consensus, opposition, parliament)
- commands
- contexts
- debug

### Plugins
- codex (knowledge crystallization, paradigms, frameworks)
- strands-tools (Python tools)
- gemini-executor
- workflow-orchestrator
- jared-intelligence
- eeps-system
- constitutional-ai
- constitutional-framework
- vision
- wiggum-marketer
- writer
- devshop
- dotfiles-dashboard
- trending
- tmux-orch
- @homie

### Context
- context patterns (systematic-opposition, figure-storming, agile-scrum)
- context schemas (stream-router, flowmind-types)
- entity types (workspace, folder, project, epic, task, portfolio, ticket, emergent)
- agent templates (EEPS personalities, strategic, analytical, creative)
- lev-self proposals

### Workshop
- workshop (POC/skill dev)
- plugins/workshop/contexts

### Planning / PM (from .lev and beads)
- FlowMind chat/voice reconciliation
- Deterministic orchestration validation
- domain.verb CLI
- Protocol handlers (bd://, file://, prompt://, review-queue://)
- Agent memory spaces
- Auto-emit BD templates from FlowMind
- Overlay system (Phase 6)
- Cognee poly integration
- Lifecycle status line + Ollama
- 5-tier memory upgrade
- UX web app (wireframes, task graph)
- Trace/store, exec traces, snapshots
- Validation gates (UBS, lev-code-review)

### Apps
- max
- shift-app
- video
- landing-astro
- auth-proxy
- universal-preview
- old-dash

---

## 10. Recommended Next Steps

1. **Promote critical specs** — FlowMind chat/voice reconciliation, lifecycle statusline → docs/specs/
2. **Add core design docs** — exec, legos, mesh, cdo, validation
3. **Plugin index** — Single docs page listing all plugins with one-line descriptions
4. **Context system doc** — Document context/, schemas, patterns
5. **App index** — Document apps/ surface
6. **Reconcile handoff content** — Mine .lev/pm/handoffs for decisions to promote

---

**END**

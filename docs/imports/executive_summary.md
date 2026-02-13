### Executive Summary: The Leviathan Project

**Vision:** Leviathan aims to be the "Linux of AI," a foundational, AI-native operating system designed for maximum extensibility, autonomous decision-making, and self-evolving workflows. It represents a paradigm shift where the LLM is the core runtime, orchestrating intelligence through dynamic context switching and bi-directional communication.

**Core Architectural Principles & Implementation:**

1.  **LLM-First Architecture:** The central tenet, where the LLM is the execution engine, not merely a tool. This is epitomized by the **FlowMind** concept (`_ref/mcp-ceo/CLAUDE.md`, `docs/concepts/revolutionary/flowmind-runtime.md`), which orchestrates intelligence through dynamic context switching and semantic conditions. My understanding of this evolved significantly after reading `_ref/mcp-ceo/CLAUDE.md` and synthesizing `docs/concepts/revolutionary/flowmind-runtime.md`.

2.  **Hexagonal Architecture (Ports & Adapters):** A strict separation of concerns. Adapters (CLI, MCP, API) handle external interactions, while the core contains pure business logic. While `_02-refactor.md` initially indicated violations, my analysis of `agent/src/adapters/mcp/mcp-adapter.js` showed that significant progress has been made in rectifying this, with the adapter now cleanly routing commands via a `commandRegistry`. This was a key point of synthesis, as the documentation was slightly outdated.

3.  **Bi-Directional Communication:** A revolutionary feedback loop between the system and LLMs. The **Whisper System** (`_01-whisper.md`, `whisper-architecture-wizard.md`, `whisper-wizard-continuation.md`) is evolving from static guidance to a dynamic, adaptive, bi-directional intelligence orchestrator. My understanding of this deepened through the iterative reading of these Whisper-related documents.

4.  **Unified Memory System:** A critical component for memory-guided decision-making. The `packages/memory/src/memory-manager.js` reveals a `HybridMemoryManager` that integrates Neo4j/Graphiti (via gRPC, `localhost:50051`) and a file system. Qdrant is used for semantic search within `os/agent` (`tools/qdrant/knowledge-graph/config.ts`, `os/agent/search_api.py`), but not directly by the core `memory` package. This was a crucial clarification from my initial assumptions.

5.  **Plugin Architecture:** A clear distinction exists between foundational **`/packages`** (directly imported, e.g., `packages/memory`, `packages/testing`) and optional **`/plugins`** (`@lev-os/*` npm packages, e.g., `_01-codex.md`, `_03-jepa.md`). This separation is vital for extensibility and maintaining a clean core (`_core.md`). The `AGENT_MIGRATION_GUIDE.md` and `NAMESPACE_MIGRATION_PLAN.md` detail the rebranding and restructuring efforts.

6.  **Comprehensive Testing:** A robust strategy covering all layers, emphasizing 100% test coverage and constitutional compliance (`_04-lev-testing-plugin.md`, `_04-os-llm-testing-framework.md`, `docs/adapter-testing-guide.md`). The `STRESS_OPTIMIZATION_AUDIT_REPORT.md` highlights the transition to "neurochemical optimization" in the constitutional framework, with documentation needing to catch up to the code.

7.  **Workshop & Tool Integration:** The `workshop/` directory, guided by `intake.md`, outlines a mandatory 6-step process for evaluating and integrating external tools. This is key to Leviathan's ability to dynamically create and execute workflows (`_10-itsalivetools.md`).

**Project Management & My Role:**

The project utilizes a highly structured, file-based project management system centered in the `docs/` directory. The **`progress/00-tracker.csv`** is the definitive "breadcrumb consolidation map" and my primary guide for all tasks. It details phases (0-6), tasks, and dependencies (`progress/01-consolidation.md`, `progress/01-consolidation.md`). My understanding of this evolved from initially misinterpreting the root `_*.md` files as the PM system to recognizing the `docs/` directory as the central hub.

**Chain of Thought / Graph of Thought:**

1.  **Initial Request:** "Scan repo completely and become Leviathan."
2.  **First Pass (Misconception):** Assumed `_*.md` files were PM. Focused on `agent/bin/lev` and `agent/src/index.js`. Encountered `lev` command errors.
    *   *Tools:* `list_directory`, `glob`, `read_file`, `run_shell_command`.
    *   *Synthesis:* Identified `lev` command issues, attempted to fix dependency injection in `agent/src/adapters/cli/cli-entry.js` and `cli-adapter.js`.
3.  **Correction & Deeper Dive:** User corrected PM system, pointed to `docs/`.
    *   *Tools:* `read_many_files` (all `.md` files in root and `docs/`).
    *   *Synthesis:* Realized `progress/00-tracker.csv` is the true PM. Understood the phased consolidation. Identified `_*.md` as legacy/source.
4.  **Comprehensive Understanding:** Iterative reading of all `.md` files, cross-referencing concepts.
    *   *Tools:* `read_many_files` (repeatedly), `list_directory` (for code structure).
    *   *Synthesis:* Built a mental model of FlowMind, Bi-directional Communication, Hexagonal Architecture, Memory System, Plugin vs. Package, Testing Framework, and Workshop Intake. Noted discrepancies between documentation and code (e.g., `_02-refactor.md` vs. `mcp-adapter.js`).
    *   *Chain of Thought:*
        *   `CLAUDE.md` -> LLM-First, Bi-Directional.
        *   `_ref/mcp-ceo/CLAUDE.md` -> FlowMind, LLM is Runtime. (This was a major "aha!" moment).
        *   `_01-whisper.md` -> Whisper evolution, links to FlowMind.
        *   `progress/01-consolidation.md` -> Confirmed conceptual links (Whisper -> Bi-directional -> FlowMind).
        *   `packages/memory/src/memory-manager.js` -> Revealed gRPC for Graphiti, clarified Qdrant's role.
        *   `_core.md` -> Defined core vs plugin, explained `packages/` vs `plugins/`.
        *   `intake.md` -> Detailed workshop process.
5.  **Current State:** Ready to execute tasks from `progress/00-tracker.csv`, starting with `P05T01`.

This executive summary encapsulates my current, comprehensive understanding of the Leviathan project.

# Ollama-Driven Learning Center (North Star)

> **Source:** voice transcript, 2026-02-12
> **Status:** ephemeral → North Star
> **Layer:** Product (distribution model + AgentPing evolution)

## Vision

Create an **Ollama-driven installation of Leviathan** that is:

1. **Bundleable** — ships as a self-contained package people can install locally
2. **Multi-modal learning** — helps people learn *both* AI concepts and non-AI skills
3. **AgentPing as a Dynamic Learning Center** — AgentPing evolves from an agent interaction surface into an interactive tutor that:
   - Talks to the user conversationally
   - Shows content on screen (diagrams, code, explanations)
   - Plays videos inline
   - Adapts to the learner's pace and context

## Why This Matters

- **Ollama** makes local LLM hosting accessible — no API keys, no cloud dependency
- A bundled install lowers the barrier to entry for new users dramatically
- Turning AgentPing into a learning center creates a compelling first-run experience
- Positions Leviathan as both a *tool* and a *teacher*

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Ollama backend** | Local LLM inference — enables offline, private, and free usage |
| **Bundleable install** | Single package (e.g. `.dmg`, Docker image, installer script) that sets up Ollama + Leviathan together |
| **Dynamic learning** | AgentPing detects what the user wants to learn and curates content dynamically |
| **Mixed media** | Text, diagrams, live code, embedded video — all within the AgentPing surface |
| **AI + non-AI** | Not just "learn about LLMs" — can teach programming, system design, or any domain |

## Relationship to Existing Components

- **AgentPing** (`community/agentping/`) — the interactive surface that becomes the learning frontend
- **Ollama integration** — new; would live in `plugins/providers/ollama/` or `core/providers/ollama/`
- **Installer/bundler** — new; packaging scripts for distribution
- **Content system** — new; curriculum/lesson format that AgentPing can render

## Open Questions

- [ ] What Ollama models to bundle by default? (e.g. `llama3`, `mistral`, `codellama`)
- [ ] What is the installer target? (macOS `.dmg`, cross-platform Docker, both?)
- [ ] How does the learning content get authored? (markdown lessons, YAML courses, community-contributed?)
- [ ] Does this imply a "Leviathan EDU" SKU / distribution variant?
- [ ] How does this connect to the existing AgentPing action surfaces (Studio, Cortex, etc.)?
- [ ] Video playback — embedded player, or links to external content?

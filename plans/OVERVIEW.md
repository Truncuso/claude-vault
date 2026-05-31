# claude-vault — Vault-Centric Claude Code Plugin System

**Date**: 2026-05-14
**Author**: Christoph Unger
**Status**: redesign (supersedes qmd-obsidian)

---

## Executive Summary

**claude-vault** is a Claude Code plugin that transforms Obsidian vaults into self-contained AI-assisted knowledge environments. Instead of a global plugin in `~/.claude/` managing vaults remotely, the `.claude/` directory lives **inside each vault** — skills, agents, memory, and QMD index evolve with the vault's context.

**Key insight:** Different vaults serve different purposes (daily work, research, learning). The plugin detects vault intent via an interactive setup skill, installs only the relevant skills, configures QMD scoped to that vault, and ships Templater templates calibrated to the vault type.

**Architecture pivot from qmd-obsidian:** The original design treated `~/.claude/` as the integration hub with a global vault registry. This created artificial coupling between vaults and blocked per-vault evolution. The new design treats each vault as an independent AI environment.

---

## Vault Types

| Type | Purpose | Skills | Status |
|------|---------|--------|--------|
| **Working Vault** | Daily notes, project tracking, meetings, raw captures, general knowledge | vault-search, obsidian-note, daily-note, ingest-content, recall, sync-sessions + working-vault tier | WP3-WP5 |
| **Research Vault** | Paper wiki, literature surveys, experiments, adversarial review, paper writing | Working set + ARIS integration (research-wiki, auto-review-loop, paper-writing, idea-discovery, lit-survey) | WP7 (draft) |
| **Learning Vault** | Courses, structured learning, book notes, concept mapping | Working set + course-tracker, book-ingest, concept-mapper, spaced-repetition | WP8 (draft) |

---

## Three-Layer Access Model + LLM

| Layer | Tool | Requires Obsidian? |
|-------|------|-------------------|
| Search | QMD per vault (BM25 + vector) | No |
| CRUD | mcpvault per vault (14 tools) | No |
| Live Ops | Obsidian CLI (search, create, daily:append) | Yes |
| Local LLM | rawveg/ollama-mcp (14 tools) | No (Ollama must run) |

---

## Installation & Setup

**Dual support:**

| Method | What Happens |
|--------|-------------|
| Marketplace: `claude plugins install claude-vault` | Skills/hooks/agents loaded. Interactive `setup` skill runs on first session — detects vault state, grills user about intent, configures .obsidian/ plugins, bootstraps .claude/, installs templates, indexes QMD. Re-runnable. |
| Manual: `git clone` + `./setup.sh` | Direct vault bootstrap. Self-contained, no ~/.claude/ dependency. |

**Setup skill** is the core onboarding experience — LLM-powered interactive conversation that adapts based on vault intent. Default setups available for quick-start.

---

## Directory Structure (within vault)

```
.claude/
├── CLAUDE.md               # Vault-specific agent instructions
├── settings.json           # MCP configs (QMD, mcpvault, Ollama MCP)
├── hooks/hooks.json        # SessionStart, FileChanged, PostCompact, Stop
├── monitors/monitors.json  # QMD daemon health
├── skills/
│   ├── general/            # All vault types (6 skills)
│   └── <vault-type>/       # Type-specific skills
├── scripts/                # Shell/Python tooling
├── templates/              # Templater templates (installed to vault's template folder)
├── prompts/
│   └── styles/             # Note style profiles (vault-local, user-editable)
└── config/
    └── config.env.example  # Vault-local config

Plugin repo also ships:
  prompts/                  # Ingestion prompt templates (versioned, shipped with plugin)
  templates/                # Templater templates (copied to vault on setup)
```

**Note on agents:** claude-vault uses Claude Code's skill system — skills ARE the agent layer. The `settings.json` `agent` path is a hook for Claude Code's built-in agent routing. No separate agent definitions are needed; each skill's `SKILL.md` frontmatter provides the agent's instructions, allowed tools, and routing. This follows the same pattern as claude-obsidian and ARIS. If custom sub-agents are needed later, they can be added under `.claude/agents/`.

**Note on prompt file locations:** Two distinct categories exist:
- **Style profiles** (`.claude/prompts/styles/`): Vault-local, user-editable YAML. Controls note voice/formality/features.
- **Ingestion prompt templates** (`prompts/` in plugin repo): Versioned, shipped with plugin. Used by LangChain pipelines for content extraction/summarization. Copied to vault during setup.

---

## Work Packages

| WP | Title | Status | Depends On |
|----|-------|--------|------------|
| WP0 | Repository Setup, Rename & Cleanup | pending | — |
| WP1 | Foundation — Per-Vault QMD Setup | pending | WP0 |
| WP2 | MCP Integration — mcpvault + CLI + Ollama | pending | WP1 |
| WP3 | General Skills (All Vault Types) | pending | WP2 |
| WP4 | Working Vault Skills + Templater Templates | pending | WP3 |
| WP5 | Configurable Note Styles & Anti-Pattern Filter | pending | WP3 |
| WP6 | Agentic Ingestion — LangChain/LlamaIndex | pending | WP3, WP5 |
| WP7 | Research Vault — ARIS Full Integration | draft | WP6 |
| WP8 | Learning Vault | draft | WP5 |

---

## Dependency Graph

```
WP0 → WP1 → WP2 → WP3 → WP4 (Working Vault)
                    ↓
                    WP5 (Styles) → WP6 (Ingestion) → WP7 (Research/ARIS)
                                                    → WP8 (Learning)
```

WP3+WP5 can partially parallelize. WP4 and WP5 are independent after WP3.

---

## Key Decisions

| Decision | Rationale | Date |
|----------|-----------|------|
| Vault-centric (`.claude/` inside vault) over global (`~/.claude/`) | Each vault evolves independently. Skills/memory/QMD scoped per vault. Cleaner architecture. | 2026-05-14 |
| Dual install: marketplace + manual | Marketplace for skill loading; setup script for vault bootstrapping. Interactive setup skill bridges both. | 2026-05-14 |
| Interactive setup skill as core onboarding | LLM-powered conversation adapts to vault intent. Re-runnable. Better than static config. | 2026-05-14 |
| rawveg/ollama-mcp over custom Ollama MCP | 155 stars, 14 tools, full Ollama SDK. Battle-tested. No need to build. | 2026-05-14 |
| ARIS integration deferred to WP7 | 65 skills, Codex dependency. Needs scoping before full integration. Core subset first. | 2026-05-14 |
| Note style profiles configurable per vault | Different vault types need different voices. Style system architected in WP5, tuned per vault. | 2026-05-14 |

---

## Reference Sources

### Core Tools
- [QMD](https://github.com/tobi/qmd) — local markdown search (BM25 + vector)
- [mcpvault](https://github.com/bitbonsai/mcpvault) — MCP-native vault CRUD
- [rawveg/ollama-mcp](https://github.com/rawveg/ollama-mcp) — Ollama MCP server (14 tools)
- [Obsidian CLI](https://obsidian.md/cli) — search, create, daily, read, eval

### Reference Repos (Explored)
- [claude-obsidian](https://github.com/Truncuso/claude-obsidian) — wiki vault (hot cache, delta tracking, feature detection, 11 skills)
- [ARIS](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep) — ML research automation (65 skills, research-wiki, auto-review-loop, integration contracts)
- [avoid-ai-writing](https://github.com/conorbronsdon/avoid-ai-writing) — AI writing detection (36 patterns, 109-entry word table, 6 context profiles)
- [obsidian-skills](https://github.com/kepano/obsidian-skills) — Agent Skills for Obsidian (5 skills, 3 interaction modes)
- [personal-os-skills](https://github.com/ArtemXTech/personal-os-skills) — recall + sync-claude-sessions

### Claude Code Docs
- [Plugins](https://code.claude.com/docs/en/plugins) | [Hooks](https://code.claude.com/docs/en/hooks-guide) | [Monitoring](https://code.claude.com/docs/en/monitoring-usage)

### Blog Posts
- ["Grep Is Dead"](https://artemxtech.substack.com/p/grep-is-dead-how-i-made-claude-code) — Artem Zhutov, 2026-03-02: QMD + /recall + Obsidian vault stack, 700 sessions in 3 weeks, semantic search finds 4/5 results without exact keywords
- [Obsidian + Claude Code Integration Guide](https://blog.starmorph.com/blog/obsidian-claude-code-integration-guide) — Kevin Lee / StarMorph: 5 strategies, QMD + session sync achieves 60%+ token reduction, "Agents read, humans write"
- [Claude Code Doesn't Index Your Codebase](https://vadim.blog/claude-code-no-indexing) — Vadim, 2026-03: Claude Code uses Glob→Grep→Read→Explore hierarchy, Boris Cherny confirms early RAG abandoned, QMD fills the indexing gap

### Plugin Target
- **Repo**: [Truncuso/claude-vault](https://github.com/Truncuso/claude-vault)
- **Local**: `/media/christoph/Samsung_Evo990/Projects/00_AI/00_Workflows/claude-vault/`

---

## Archived Plans

Stale plans from the `qmd-obsidian` global-plugin era are archived at `plans/_archive/`. These reference `~/.claude/` paths, global vault registry, and a superseded architecture.

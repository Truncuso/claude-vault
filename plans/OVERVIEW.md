# QMD + Obsidian Vault Integration System

**Date**: 2026-05-13
**Updated**: 2026-05-13
**Author**: Christoph Unger
**Status**: specified

---

## Executive Summary

Integrate QMD (local markdown semantic search engine) with Obsidian vault management into Claude Code. The user has a mature `.claude` configuration (32 skills, 42 agents, SDD workflow) but QMD has 0 indexed documents and no Obsidian MCP access exists. This plan delivers a portable, env-var-driven three-layer access model (QMD for search, mcpvault for CRUD, Obsidian CLI for live ops), manages multiple vaults, evolves the draft agentic note ingestion project from smolagents to LangChain/LlamaIndex, and enforces verifiable contracts at every layer.

**Key insight:** Three-layer access model — QMD handles all semantic/keyword search (saves tokens by finding context before loading files), mcpvault provides MCP-native CRUD without needing Obsidian running, and Obsidian CLI handles template-based note creation when the app IS running.

---

## Active Work Packages

| WP | Title | Severity | Status | Impact |
|----|-------|----------|--------|--------|
| WP1 | Foundation — Environment & Path Conventions | HIGH | specified | All files — fixes hardcoded paths, establishes env var system |
| WP2 | QMD Configuration — Collections, Indexing, Daemon, Auto-Update | HIGH | specified | New: 4 scripts, 2 systemd units, 1 hook, 1 MCP config |
| WP3 | Obsidian MCP Access — mcpvault + CLI | HIGH | specified | New: MCP configs per vault, CLI wrapper script |
| WP4 | Multi-Vault Management | MEDIUM | specified | New: 2 scripts, template dir, test vault creation |
| WP5 | Skills & Templates | MEDIUM | specified | New: 5 skills, 2 skill modifications |
| WP6 | Agentic Ingestion — LangChain/LlamaIndex Migration | HIGH | specified | Refactor: smolagents→LangChain, +podcast pipeline, +evaluator |

---

## Archived / Closed / Deleted

| WP | Outcome |
|----|---------|
| — | — |

---

## Corrections Log

| Previous Claim | Corrected Finding |
|----------------|-------------------|
| Build custom Python MCP bridge for obsidian-local-rest-api | Use mcpvault (bitbonsai/mcpvault) — MCP-native, no bridge needed, 14 tools |
| Keep smolagents framework | Migrate to LangChain/LlamaIndex per user's explicit choice |
| Auto-embed: both `qmd update` and `qmd embed` via inotify | Split: `qmd update` via inotify (fast file scan), `qmd embed` via manual/cron (expensive ML). Both commands are global — no `--collection` flag exists. |
| WP6 depends on WP5 | WP5 and WP6 are independent — both depend on WP2+WP3, no mutual dependency |

---

## Execution Strategy

```
WP1 (Foundation)
  |
  +---> WP2 (QMD) -----> WP4 (Multi-Vault)
  |        |                 |
  +---> WP3 (mcpvault+CLI) --+
           |
           +---> WP5 (Skills)
           |
           +---> WP6 (Ingestion)
```

WP2 and WP3 are parallelizable (no mutual dependencies). WP4 depends on WP2 (vault creation triggers QMD collection setup). WP5 depends on WP2+WP3 (skills use QMD and mcpvault tools). WP6 depends on WP2+WP3 (LangChain pipelines use QMD for search, mcpvault for writing output). WP5 and WP6 are independent — they can run in parallel.

---

## Key Decisions

| Decision | Rationale | Date |
|----------|-----------|------|
| Three-layer access: QMD + mcpvault + Obsidian CLI | QMD = search (token-saving), mcpvault = CRUD (MCP-native, no Obsidian needed), CLI = live ops (templates, UI). Complementary, not redundant. | 2026-05-13 |
| mcpvault over obsidian-local-rest-api bridge | mcpvault is MCP-native (zero bridge code), works without Obsidian running, 14 tools. REST API plugin stays as fallback. | 2026-05-13 |
| QMD daemon: systemd + SessionStart hook | systemd for persistence across reboots; SessionStart hook as belt-and-suspenders fallback | 2026-05-13 |
| Auto-update split strategy: `qmd update` via inotify + `qmd embed` via manual/cron | `qmd update` is fast (file scan, safe to automate). `qmd embed` is expensive (global ML inference on ALL docs) — manual or daily cron. Both commands are global (no `--collection` flag exists). | 2026-05-14 |
| LangChain/LlamaIndex for ingestion | User's explicit choice. Better ecosystem compatibility, structured output, evaluation tooling. | 2026-05-13 |
| Env-var-based paths, no hardcoded absolute paths | Enables portability across workstations. `config.env` is the single source of workstation-specific config. | 2026-05-13 |

---

## Reference Sources

All resources, documentation, repos, and blog posts referenced during planning. These are the canonical external sources for this project.

### Core Tools

| Source | URL | Role |
|--------|-----|------|
| QMD GitHub | [https://github.com/tobi/qmd](https://github.com/tobi/qmd) | QMD — local markdown semantic search engine (BM25 FTS5 + vector via sqlite-vec + node-llama-cpp) |
| QMD plugin marketplace | `~/.claude/plugins/marketplaces/qmd/` | QMD Claude Code plugin (v0.9.0), MCP server via HTTP daemon |
| QMD CLAUDE.md | `~/.claude/plugins/marketplaces/qmd/CLAUDE.md` | QMD's own agent instructions — contains prohibition on auto-running commands |
| mcpvault | [https://github.com/bitbonsai/mcpvault](https://github.com/bitbonsai/mcpvault) | MCP-native Obsidian vault CRUD (`npx @bitbonsai/mcpvault`), 14 tools, headless |
| mcp-obsidian (evaluated, not chosen) | [https://github.com/MarkusPfundstein/mcp-obsidian](https://github.com/MarkusPfundstein/mcp-obsidian) | Alternative MCP server for Obsidian — evaluated, mcpvault preferred |
| Obsidian CLI docs | [https://obsidian.md/cli](https://obsidian.md/cli) | Obsidian CLI reference (search, create, daily:append, read, eval) |

### Claude Code Plugin Documentation

| Source | URL | Role |
|--------|-----|------|
| Claude Code Plugins | [https://code.claude.com/docs/en/plugins](https://code.claude.com/docs/en/plugins) | Plugin system: manifest, skills, hooks, monitors, bin/ |
| Claude Code Hooks Guide | [https://code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide) | Lifecycle hooks: SessionStart, FileChanged, PostCompact, Stop |
| Claude Code Monitoring | [https://code.claude.com/docs/en/monitoring-usage](https://code.claude.com/docs/en/monitoring-usage) | Background monitors, usage tracking |

### Inspiration & Prior Art

| Source | URL/Path | Role |
|--------|----------|------|
| Artem Zhutov blog post | [https://x.com/ArtemXTech/status/2028330693659332615](https://x.com/ArtemXTech/status/2028330693659332615) | Inspiration: personal OS skills, recall, sync-claude-sessions workflow |
| personal-os-skills | `/media/christoph/Samsung_Evo990/Projects/00_AI/00_Workflows/personal-os-skills/` | Source repo for `recall` and `sync-claude-sessions` skills — 7 skills total |
| agentic_note_ingestion (draft) | `Obsidian_Work_Vault/System/000_Scripts/agentic_note_ingestion/` | Existing Python ingestion project (smolagents → LangChain migration target) |
| Obsidian Work Vault | `/media/christoph/M2.SSD1/000_Vaults/Obsidian_Work_Vault/` | Primary vault — 26 plugins, Templater templates, structured folders |

### Plugin Target

| Item | Value |
|------|-------|
| Plugin repo | `Truncuso/qmd-obsidian` |
| Local path | `/media/christoph/Samsung_Evo990/Projects/00_AI/00_Workflows/qmd-obsidian/` |
| GitHub | SSH: `git@github.com:Truncuso/qmd-obsidian.git` |

---

## Acceptance Criteria

- [ ] `grep -r '/home/' ~/.claude/scripts/` returns 0 hardcoded paths (except config.env comments)
- [ ] QMD collections populated: `qmd status --json` shows document_count > 0 for work-vault
- [ ] QMD daemon running: `curl localhost:8181/health` → 200
- [ ] mcpvault functional: `read_note` returns content, `write_note` creates file visible in Obsidian
- [ ] Vault switching: `vault switch test` updates $OBSIDIAN_ACTIVE_VAULT and QMD scope
- [ ] All 5 new skills load and execute correctly
- [ ] Ingestion pipeline produces evaluated, structured markdown notes from YouTube/PDF/podcast sources
- [ ] All verification matrix invariants (19 total) pass
- [ ] Test suites: unit, integration, E2E all green

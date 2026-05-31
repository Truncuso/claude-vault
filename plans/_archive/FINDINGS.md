# Detailed Findings — QMD + Obsidian Vault Integration

**Date**: 2026-05-13
**Updated**: 2026-05-13
**Analysis Method**: Exploration via parallel sub-agents: (1) .claude directory audit, (2) external vaults + personal-os-skills inspection, (3) dotfiles/claude repo analysis. Architectural evaluation via Plan agent.

---

## Corrections Summary

| Original Claim | Corrected Finding |
|----------------|-------------------|
| Need to install Obsidian MCP server (mcp-obsidian npm package) | obsidian-local-rest-api plugin already installed on port 27124. mcpvault (`@bitbonsai/mcpvault`) is a better fit: MCP-native via npx, no plugin needed, 14 tools, token-optimized. |
| Build a custom Python MCP bridge for obsidian-local-rest-api | Unnecessary — mcpvault provides the same CRUD capabilities as MCP tools natively without a bridge server to build/maintain. Keep REST API as fallback only. |
| Obsidian CLI not useful for programmatic access | It IS useful — search, create from templates, daily note append, eval. But requires Obsidian running. Complementary to mcpvault for headless ops. |
| Keep smolagents for ingestion | User explicitly chose LangChain/LlamaIndex. Migration needed but existing architecture (JSON contracts, RuntimeConfig, extraction services) is reusable. |
| Single vault is sufficient initially | User explicitly wants multi-vault from the start. Vault registry + creation scripts needed in WP4. |

---

## Finding 1: QMD is Installed but Completely Unconfigured

### 1.1 Current State

QMD v0.9.0 is installed at `/home/christoph/.nvm/versions/node/v20.17.0/bin/qmd`, the plugin is enabled in settings.json, and the MCP server tools are available via `mcp__plugin_qmd_qmd__*`. However, the SQLite index at `~/.cache/qmd/index.sqlite` contains 0 collections, 0 documents, 0 vectors. No collections have been defined. No daemon is running. The system-reminder message confirms "0 markdown documents."

### 1.2 Structural Differences

| Aspect | Current | Expected |
|--------|---------|----------|
| Collections | 0 | 2+ (work-vault, sessions; more per vault) |
| Documents indexed | 0 | 1000+ (all .md files in work vault) |
| Vector embeddings | 0 | Generated for all indexed documents |
| Daemon | Not running | Running persistently on port 8181 |
| Auto-update | None | inotify-based watcher + git hooks |
| Context descriptions | None | Per-sub-path context for relevance boosting |

### 1.3 Confirmed Gap

QMD's core value proposition — saving tokens by finding relevant context before loading files — is completely unused. Every search in the current setup requires loading entire files into context. This is the highest-impact fix in the entire plan.

**Impact**: Every Claude Code session wastes context window tokens by loading unnecessary or tangentially-relevant vault content. With QMD semantic search, the agent can retrieve only the 2-3 most relevant documents instead of scanning entire folders.

### 1.4 Relevant Code Paths

| File | Line(s) | Role |
|------|---------|------|
| `/home/christoph/.cache/qmd/index.sqlite` | — | Empty SQLite index (72KB file, no data) |
| `/home/christoph/.claude/settings.json` | — | Plugin enabled, marketplace configured |
| `/home/christoph/.claude/plugins/cache/qmd/qmd/0.1.0/` | — | QMD plugin installation |
| `~/.nvm/versions/node/v20.17.0/bin/qmd` | — | QMD binary |

---

## Finding 2: No Obsidian Access from Claude Code

### 2.1 Current State

Claude Code has no MCP tools for Obsidian vault operations. The obsidian-local-rest-api plugin is installed and configured (port 27124, API key present), but there's no MCP bridge. The Obsidian CLI exists at `/usr/bin/obsidian` but is the Electron app launcher — no wrapper exists for MCP integration. The personal-os-skills (recall, sync-claude-sessions) access vaults through direct filesystem paths, not through any MCP layer.

### 2.2 Confirmed Gap

Three access patterns are needed:
1. **Headless CRUD** (no Obsidian running) — mcpvault fills this gap. Works on vault directory directly.
2. **Live operations** (Obsidian running) — Obsidian CLI for template rendering, UI commands.
3. **Search** (always available) — QMD for semantic/keyword search (fills this gap once WP2 is done).

### 2.3 Relevant Code Paths

| File | Line(s) | Role |
|------|---------|------|
| `Obsidian_Work_Vault/.obsidian/plugins/obsidian-local-rest-api/` | — | Installed REST API plugin |
| `/usr/bin/obsidian` | — | Obsidian desktop app / CLI launcher |
| `mcp-configs/mcp-servers.json` | — | 26 MCP servers, 0 Obsidian-related |

---

## Finding 3: agentic_note_ingestion Has Strong Architecture, Wrong Framework

### 3.1 Current State

The project at `Obsidian_Work_Vault/System/000_Scripts/agentic_note_ingestion/` is well-structured:
- Clean package layout (`src/ingestion/{common,youtube,pdf}/`)
- JSON contract conventions for all outputs
- `RuntimeConfig` dataclass with env var resolution
- Formal ARCHITECTURE.md with mathematical optimization objective
- Verification matrix with 8 invariants
- Templater templates that call Python orchestrators from within Obsidian
- Tests with smoke test coverage

However:
- Uses smolagents (user wants LangChain/LlamaIndex)
- Prompts are embedded in Templater JavaScript templates, not versioned
- No model discovery/selection
- No output quality evaluation
- No podcast pipeline
- No unified CLI

### 3.2 Confirmed Gap

The extraction layer (services.py for YouTube and PDF) is reusable. The orchestration layer needs framework migration. Missing capabilities (model discovery, prompt registry, evaluator, podcast pipeline, CLI) are additive — they don't require rewriting existing code.

**Impact**: The ingestion pipeline is the highest-leverage workflow in the system — it transforms raw content (videos, papers, podcasts) into structured, searchable, linkable knowledge. Without these additions, it remains a draft project.

### 3.3 Relevant Code Paths

| File | Line(s) | Role |
|------|---------|------|
| `agentic_note_ingestion/ARCHITECTURE.md` | — | Formal architecture spec |
| `agentic_note_ingestion/src/ingestion/common/config.py` | — | RuntimeConfig (reusable) |
| `agentic_note_ingestion/src/ingestion/youtube/orchestrator.py` | — | smolagents → needs LangChain refactor |
| `agentic_note_ingestion/src/ingestion/youtube/services.py` | — | Extraction (reusable) |
| `agentic_note_ingestion/src/ingestion/pdf/orchestrator.py` | — | smolagents → needs LangChain refactor |
| `agentic_note_ingestion/src/ingestion/pdf/services.py` | — | Extraction/chunking (reusable) |
| `Obsidian_Work_Vault/System/00_Templates/YT_Local_Ingestion_v2.md` | — | Templater template (backward compat required) |
| `Obsidian_Work_Vault/System/00_Templates/PDF_Agent_Ingestion.md` | — | Templater template (backward compat required) |

---

## Finding 4: Hardcoded Paths Block Portability

### 4.1 Current State

Multiple files contain hardcoded paths referencing `/home/cunger/` (a different user's home directory). The hooks.json and settings.json both have these. These break when the configuration is synced to a machine where the home directory is `/home/christoph/`. No central env var file exists.

### 4.2 Confirmed Gap

The configuration is not portable. Every new workstation requires manual path fixes. Without WP1, none of the subsequent phases can be implemented portably.

**Impact**: Any script, hook, or MCP config that references vault paths will break on a different machine. The env var system in WP1 is a hard prerequisite for all other work.

### 4.3 Relevant Code Paths

| File | Line(s) | Role |
|------|---------|------|
| `~/.claude/settings.json` | — | Contains `/home/cunger/` references |
| `~/.claude/hooks/hooks.json` | — | Multiple hardcoded `/home/cunger/` paths |

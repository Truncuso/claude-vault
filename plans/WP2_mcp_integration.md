# WP2: MCP Integration — mcpvault + Obsidian CLI + Ollama MCP

**Status**: pending
**Severity**: HIGH
**Created**: 2026-05-14
**Depends on**: WP1

---

## Problem

The vault has QMD search but no CRUD access, no live Obsidian operations, and no local LLM inference. Three MCP servers need configuration: mcpvault (headless vault CRUD), Obsidian CLI (live template-based ops), and Ollama MCP (local model access for review, embedding, drafting).

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1 | Add mcpvault MCP config to `settings.json` | pending |
| 2 | Create `scripts/obsidian/obsidian-cli.sh` — wrapper with JSON output | pending |
| 3 | Add Ollama MCP config to `settings.json` | pending |
| 4 | Document three-layer + LLM access in `CLAUDE.md` template | pending |

---

## Step-by-step

### Step 1 — mcpvault MCP Config

Added to `.claude/settings.json` per vault. One mcpvault instance per vault directory.

```json
{
  "mcpvault-<vault-name>": {
    "command": "npx",
    "args": ["-y", "@bitbonsai/mcpvault@1.0.0", "<vault-path>"],
    "description": "CRUD access to <vault-name>: read/write/patch/delete notes, frontmatter, tags, search"
  }
}
```

**Security note:** Pin to a specific version (not `@latest`) — mcpvault has full vault CRUD. Supply-chain risk if left unpinned.

### Step 2 — obsidian-cli.sh

Wrapper around `/usr/bin/obsidian` CLI. All commands output JSON. Graceful fallback when Obsidian is not running.

```bash
obsidian-cli search <query> [--vault <name>]
obsidian-cli create <note-name> [--template <name>]
obsidian-cli daily:append <content>
obsidian-cli read <path>
obsidian-cli eval <js-expression>
```

**Fallback contract:**
```json
{"error": "Obsidian not running", "error_code": "OBSIDIAN_UNAVAILABLE"}
```

### Step 3 — Ollama MCP Config

Uses [rawveg/ollama-mcp](https://github.com/rawveg/ollama-mcp) — 14 tools, full Ollama SDK coverage. AGPL-3.0 licensed, forkable.

```json
{
  "ollama": {
    "command": "npx",
    "args": ["-y", "ollama-mcp@latest"],
    "env": {
      "OLLAMA_HOST": "http://127.0.0.1:11434"
    },
    "description": "Local LLM via Ollama: chat, generate, embed, list, pull, push, delete models"
  }
}
```

**Available tools:** `ollama_chat`, `ollama_generate`, `ollama_embed`, `ollama_list`, `ollama_pull`, `ollama_push`, `ollama_delete`, `ollama_create`, `ollama_copy`, `ollama_show`, `ollama_ps`, plus web search/fetch.

### Step 4 — CLAUDE.md Template

Documents the access model for agents working in the vault:

```markdown
## Vault Access

You have three MCP servers for this vault:

1. **QMD** (`mcp__plugin_qmd_qmd__*`) — semantic + keyword search. Use FIRST for any lookup.
2. **mcpvault** (`mcp__mcpvault-<name>__*`) — CRUD operations. Use for reading/writing notes.
3. **Obsidian CLI** (`obsidian-cli`) — template creation, daily notes. Only when Obsidian is running.
4. **Ollama MCP** (`mcp__ollama__*`) — local LLM for review, summarization, drafting.
```

---

## Verification

| ID | Test | Expected | Method |
|----|------|----------|--------|
| V2.1 | mcpvault lists 14 tools | All tools present | MCP `list_tools` |
| V2.2 | mcpvault read/write roundtrip | Written = read content | write → read → compare → delete |
| V2.3 | Obsidian CLI search returns results | Results when Obsidian running | `obsidian search "test"` |
| T2.1 | mcpvault read_note on existing file | Returns known content | Read known vault file, compare hash |
| T2.2 | mcpvault write_note visible in Obsidian | File appears in file explorer | Create note, check Obsidian UI |
| T2.3 | mcpvault get_frontmatter parses YAML | Structured frontmatter returned | Read note with known frontmatter |
| T2.4 | mcpvault search_notes BM25 | Known content ranks high | Search for unique phrase |
| T2.5 | mcpvault patch_note preserves other content | Only specified changes applied | Patch frontmatter, verify body unchanged |
| T2.6 | obsidian-cli handles Obsidian not running | Error JSON, non-zero exit, no crash | Run with Obsidian closed |
| T2.7 | obsidian-cli create uses template | Template vars resolved | Create from template, inspect output |
| T2.8 | Ollama MCP lists available models | `ollama_list` returns models | MCP `ollama_list` |
| T2.9 | Ollama MCP generates text | `ollama_generate` produces output | MCP `ollama_generate` with test prompt |

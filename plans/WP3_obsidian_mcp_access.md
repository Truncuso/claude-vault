# WP3: Obsidian MCP Access — mcpvault + CLI

**Status**: specified
**Severity**: HIGH
**Created**: 2026-05-13
**Implemented**: —
**Depends on**: WP1

---

## Problem

Claude Code has no programmatic access to Obsidian vaults. No MCP tools exist for reading/writing notes, searching within vaults, manipulating frontmatter, or creating notes from templates. The obsidian-local-rest-api plugin is installed on port 27124 but has no MCP bridge. The Obsidian CLI (`/usr/bin/obsidian`) provides live operations but needs a wrapper for JSON contract compliance.

---

## Verified Evidence

- No Obsidian-related MCP servers in `mcp-configs/mcp-servers.json` (confirmed: 28 entries, none Obsidian)
- `obsidian-local-rest-api` plugin v3.6.2 installed in work vault `.obsidian/plugins/` with API key configured
- `obsidian` binary at `/usr/bin/obsidian` — Electron app launcher with CLI subcommands (search, create, daily, read, eval)
- mcpvault (`@bitbonsai/mcpvault`) evaluated: 14 MCP tools, no plugin needed, works via `npx`, token-optimized responses
- Obsidian CLI caveat: requires Obsidian app to be running

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1 | Add mcpvault MCP config per vault to `mcp-servers.json` | pending |
| 2 | Create `~/.claude/scripts/obsidian/obsidian-cli.sh` | pending |
| 3 | Test mcpvault CRUD roundtrip | pending |

---

## Step-by-step

### Step 1 — mcpvault MCP config

One MCP server entry per vault (mcpvault serves one vault directory per instance):

```json
{
  "mcpvault-work": {
    "command": "npx",
    "args": ["-y", "@bitbonsai/mcpvault@latest", "${OBSIDIAN_WORK_VAULT}"],
    "description": "CRUD access to work vault via mcpvault: read/write/patch/delete notes, frontmatter, tags, search, directory listing"
  }
}
```

Additional vaults (test, personal) get their own entries on creation (WP4).

### Step 2 — obsidian-cli.sh

Wraps Obsidian CLI with JSON output contracts. Only works when Obsidian app is running.

```bash
# Subcommands
obsidian-cli search <query> [--vault <name>] [--json]
obsidian-cli create <name> [--template <t>] [--vault <name>] [--json]
obsidian-cli append <note> <content> [--vault <name>] [--json]
obsidian-cli read [--vault <name>] [--json]         # Current active file
obsidian-cli daily [--vault <name>]                   # Open daily note
obsidian-cli daily:append <content> [--vault <name>]  # Append to daily note

# Output contract (--json flag)
{"status": "success", "command": "search", "results": [...]}
{"status": "error", "command": "search", "error_code": "OBSIDIAN_NOT_RUNNING", "message": "Obsidian app is not running"}
```

### Step 3 — Test CRUD roundtrip

Verify mcpvault can write a note and read it back:
1. `write_note(path="test/roundtrip.md", content="# Test")` → success
2. `read_note(path="test/roundtrip.md")` → returns `# Test`
3. `delete_note(path="test/roundtrip.md")` → success

---

## Verification

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| Shell syntax | `bash -n ~/.claude/scripts/obsidian/obsidian-cli.sh` | No errors |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V2.1 | mcpvault lists 14 tools | All tools present in list | MCP `list_tools` for mcpvault-work |
| V2.2 | mcpvault read/write roundtrip | Written content matches read content | write → read → compare → delete |
| V2.3 | Obsidian CLI search returns results | Results when Obsidian is running | `obsidian search "test"` |
| T1 | mcpvault read_note returns existing vault content | Known file content matches | Read known vault file, compare |
| T2 | mcpvault write_note creates file visible in Obsidian | File appears in Obsidian file explorer | Create test note, check Obsidian UI |
| T3 | mcpvault get_frontmatter parses YAML | Returns structured frontmatter | Read note with known frontmatter |
| T4 | mcpvault search_notes finds content | BM25-ranked results | Search for known content |
| T5 | mcpvault patch_note preserves other content | Only specified changes applied, body unchanged | Patch frontmatter, verify body unchanged |
| T6 | obsidian-cli handles Obsidian not running gracefully | Error JSON, not crash | Run with Obsidian closed |
| T7 | obsidian-cli create uses template | Template variables resolved | Create from template, inspect output |

---

## Review

| Reviewer | Verdict | Notes |
|-----------|---------|-------|
| — | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| mcpvault one-vault-per-instance limitation | LOW | Documented; one MCP entry per vault |
| Obsidian CLI requires app running | MEDIUM | Documented; mcpvault handles headless operations |

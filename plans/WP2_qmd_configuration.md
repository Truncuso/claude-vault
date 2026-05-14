# WP2: QMD Configuration — Collections, Indexing, Daemon, Auto-Update

**Status**: specified
**Severity**: HIGH
**Created**: 2026-05-13
**Implemented**: —
**Depends on**: WP1

---

## Problem

QMD v0.9.0 is installed and the MCP plugin is enabled, but the index is empty: 0 collections, 0 documents, 0 vectors. No daemon is running for persistent MCP access. No mechanism exists to keep embeddings up-to-date when vault content changes. QMD's semantic search capability (the primary value proposition for saving tokens) is completely unused.

---

## Verified Evidence

- `qmd status --json` → `{"collections": 0, "documents": 0, "vectors": 0}` (confirmed during exploration)
- `/home/christoph/.cache/qmd/index.sqlite` exists (72KB) but is empty
- No systemd user services for QMD exist under `~/.config/systemd/user/`
- No SessionStart hook for QMD daemon lifecycle
- QMD CLAUDE.md (bundled with plugin): "Never run `qmd collection add`, `qmd embed`, or `qmd update` automatically" — this is **agent-level policy** (AI assistants must not autonomously trigger these; write commands for the user instead). It is NOT a system architecture constraint — user-configured automation (systemd watchers, cron) is fine since the user consciously set it up.

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1 | Create `~/.claude/scripts/qmd/setup-collections.sh` | pending |
| 2 | Create `~/.claude/scripts/qmd/qmd-daemon.sh` | pending |
| 3 | Create `~/.claude/scripts/qmd/auto-update.sh` | pending |
| 4 | Create `~/.config/systemd/user/qmd-daemon.service` | pending |
| 5 | Create `~/.config/systemd/user/qmd-auto-update.service` | pending |
| 6 | Create `~/.claude/scripts/qmd/contexts/work-vault.md` | pending |
| 7 | Modify `~/.claude/hooks/hooks.json` — SessionStart hook | pending |
| 8 | Modify `dotfiles/claude/mcp-configs/mcp-servers.json` — QMD HTTP config | pending |

---

## Step-by-step

### Step 1 — setup-collections.sh

Idempotent setup script. Creates QMD collections, adds context descriptions, runs initial `qmd update` + `qmd embed`.

```bash
# Collections
qmd collection add work-vault "${OBSIDIAN_WORK_VAULT}" --mask "**/*.md"
qmd collection add sessions "${HOME}/.claude/sessions" --mask "*.jsonl"

# Context descriptions (improve search relevance)
qmd context add qmd://work-vault/003_Projects "Active projects, implementation plans, architecture decisions"
qmd context add qmd://work-vault/Areas "Domain knowledge areas, research notes, reference material"
qmd context add qmd://work-vault/System "Templates, scripts, automation, system configuration"

# Initial indexing (both commands are global — operate on all collections)
qmd update
qmd embed
```

**Important:** `qmd update` and `qmd embed` do NOT accept `--collection` — they are global operations on all collections. `qmd update` is fast (file scanning). `qmd embed` is expensive (model inference on ALL documents). See Step 3 for the split automation strategy.

### Step 2 — qmd-daemon.sh

```bash
qmd-daemon start     # Start HTTP daemon on $QMD_DAEMON_PORT
qmd-daemon stop      # Kill daemon process
qmd-daemon status    # Check if running, return JSON
qmd-daemon ensure    # Start if not running (for SessionStart hook)
```

### Step 3 — auto-update.sh (fast: file scan) + embed strategy (expensive: manual/cron)

**`qmd update`** is a fast file scan — safe to automate via inotify. **`qmd embed`** is expensive model inference on ALL documents across ALL collections — should be manual or daily cron, NOT per-file-change.

**auto-update.sh** (inotify → `qmd update` only):

```bash
#!/bin/bash
# Watches all registered vaults for .md changes, runs `qmd update` (fast file scan).
# Debounce: 30s cooldown after last change before running.
# Does NOT run `qmd embed` — that is handled separately (manual or cron).

WATCH_DIRS="${OBSIDIAN_WORK_VAULT}"
# Additional vault dirs appended as vaults are registered (WP4).

inotifywait -m -r --format '%w%f' -e modify,create,delete,move \
  ${WATCH_DIRS} --include '\.md$' \
  | while read f; do
      # Debounce 30s: wait for quiet period, then qmd update
      sleep 30
      qmd update
    done
```

**Embed strategy — `qmd embed`** (manual or cron):

```
# Manual: run after major vault changes (new vault, bulk import)
qmd embed

# Cron (daily at 3am, off-peak): re-generate all embeddings
# 0 3 * * * qmd embed
```

**Rationale for split:** `qmd update` rescans files for BM25 search — fast, negligible cost, safe to automate. `qmd embed` runs the embedding model (node-llama-cpp) on every document in every collection — expensive and wasteful on every keystroke. BM25 search works without embeddings. Vector search is a bonus. The user runs `qmd embed` manually after significant vault changes, or via daily cron.

### Step 4-5 — systemd units

User-scoped systemd services:
- `qmd-daemon.service` — `ExecStart=qmd mcp --daemon --port ${QMD_DAEMON_PORT}`
- `qmd-auto-update.service` — `ExecStart=bash ~/.claude/scripts/qmd/auto-update.sh` (watches .md files, runs `qmd update` on changes)

`qmd embed` is NOT automated as a service — it is run manually or via cron (see Step 3).

### Step 7 — SessionStart hook

```json
{
  "matcher": "*",
  "hooks": [{
    "type": "command",
    "command": "bash ${CLAUDE_CONFIG_HOME}/scripts/qmd/qmd-daemon.sh ensure"
  }]
}
```

### Step 8 — MCP server config

```json
{
  "qmd-http": {
    "url": "http://localhost:${QMD_DAEMON_PORT}/mcp",
    "transport": "http",
    "description": "QMD semantic search via HTTP daemon"
  }
}
```

---

## Verification

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| Shell syntax | `bash -n ~/.claude/scripts/qmd/setup-collections.sh` | No errors |
| Shell syntax | `bash -n ~/.claude/scripts/qmd/qmd-daemon.sh` | No errors |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V1.1 | QMD collections exist, doc count > 0 | document_count > 0, collections list complete | `qmd status --json` |
| V1.2 | QMD daemon responds on port 8181 | HTTP 200 | `curl localhost:8181/health` |
| V1.3 | Auto-update triggers on .md file change | `qmd update` runs within 35s of .md change | `touch test.md` in vault, wait 35s, check update log |
| V1.4 | `qmd embed` works when run manually | Embeddings generated for all collections | Run `qmd embed` manually, verify vector count > 0 in `qmd status --json` |
| T1 | BM25 search returns ranked results | Results with scores | `qmd search "machine learning" --collection work-vault` |
| T2 | MCP search tool functional | Returns valid results | `mcp__plugin_qmd_qmd__search` with test query |
| T3 | MCP get tool retrieves document | Returns full content | `mcp__plugin_qmd_qmd__get` with known file path |
| T4 | Context descriptions improve relevance | Scoped search returns domain-relevant results | Compare scoped vs. unscoped search relevance |
| T5 | Daemon survives SessionStart re-invoke | Idempotent — no duplicate process | Call `qmd-daemon.sh ensure` twice, check single process |

---

## Review

| Reviewer | Verdict | Notes |
|-----------|---------|-------|
| — | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| `inotify-tools` package required (not installed by default) | MEDIUM | Add install step to WP1 dependencies |
| `qmd embed` is global and expensive — no per-collection scoping | MEDIUM | Documented as manual/cron operation; revisit when QMD adds `--collection` flag |
| inotify watch limit may need increase on large vaults | LOW | Document in SETUP.md; current limit 65536 is sufficient |

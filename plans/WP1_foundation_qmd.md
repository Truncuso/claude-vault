# WP1: Foundation — Per-Vault QMD Setup

**Status**: pending
**Severity**: HIGH
**Created**: 2026-05-14
**Depends on**: WP0

---

## Problem

A vault bootstrapped with `.claude/` has no QMD search. The original design created one global QMD collection for all vaults — wrong. Each vault needs its own QMD collection scoped to its content, a daemon for MCP access, and automatic re-indexing on file changes.

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1 | Create script: `scripts/lib/env.sh` — vault-relative path resolution | pending |
| 2 | Create script: `scripts/qmd/setup-collections.sh` — idempotent collection setup | pending |
| 3 | Create script: `scripts/qmd/qmd-daemon.sh` — start/stop/status/ensure | pending |
| 4 | Create `hooks/hooks.json` — SessionStart (daemon ensure), FileChanged (qmd update), Stop (freshness) | pending |
| 5 | Create `monitors/monitors.json` — QMD daemon health check | pending |
| 6 | Create `config/config.env.example` — vault-local config | pending |

---

## Step-by-step

### Step 1 — env.sh

Sourced by all scripts. Vault-relative — resolves paths from `$VAULT_ROOT` (the vault directory containing `.claude/`). No global env vars except secrets (API keys, which stay in `config.env`).

```bash
# Resolves vault root: the directory containing .claude/
VAULT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/../.." && pwd)"
source "${VAULT_ROOT}/.claude/config/config.env" 2>/dev/null || true

validate_required_vars() {
    # Checks QMD_DAEMON_PORT, OLLAMA_BASE_URL are set
}
```

### Step 2 — setup-collections.sh

Idempotent — safe to re-run. Creates QMD collection for the vault, runs initial index.

```bash
# qmd collection add <vault-name> <vault-path> --mask "**/*.md"
# qmd update       # Fast file scan (global, but only this vault's collection matters)
# qmd embed        # Expensive ML inference. Inform user, can defer.
```

**Important:** `qmd update` and `qmd embed` are GLOBAL operations (no `--collection` flag exists). The `qmd collection add` scopes what gets indexed; subsequent global commands process all collections.

### Step 3 — qmd-daemon.sh

```bash
qmd-daemon {start|stop|status|ensure}

# ensure: starts daemon if not running, no-op if already up
# Returns JSON: {"status":"running","port":8181} or {"status":"failed","error":"..."}
```

### Step 4 — hooks.json

```json
{
  "SessionStart": [
    {"command": "bash .claude/scripts/qmd/qmd-daemon.sh ensure || true", "timeout": 10000},
    {"command": "echo '{\"vault\":\"'$(basename \"$VAULT_ROOT\" 2>/dev/null || echo unknown)'\"}' || true", "timeout": 3000}
  ],
  "FileChanged": [
    {"pattern": "**/*.md", "command": "cd \"$VAULT_ROOT\" && qmd update", "mode": "async", "debounce_ms": 30000}
  ],
  "Stop": [
    {"command": "qmd status --json 2>/dev/null | python3 -c \"import sys,json; d=json.load(sys.stdin); print(f'QMD: {d.get(\\\"document_count\\\",0)} docs')\" || true"}
  ]
}
```

### Step 5 — monitors.json

```json
{
  "qmd-daemon-health": {
    "command": "while true; do curl -sf localhost:${QMD_DAEMON_PORT:-8181}/health > /dev/null 2>&1; sleep 30; done",
    "description": "Health-check QMD daemon every 30s"
  }
}
```

### Step 6 — config.env.example

Vault-local config. Git-tracked as example. Real `config.env` is gitignored.

**Environment variable semantics:**

| Variable | Set by | Used by | Purpose |
|----------|--------|---------|---------|
| `VAULT_ROOT` | Runtime (resolved by `env.sh`) | All scripts, hooks, monitors | Absolute path to vault directory containing `.claude/`. Resolved by walking up from script location. Never user-set. |
| `OBSIDIAN_VAULT_NAME` | User (in `config.env`) | Obsidian CLI (`--vault` flag) | Human-readable vault name string matching the name shown in Obsidian's vault switcher. Required for `obsidian` CLI commands. |
| `QMD_DAEMON_PORT` | User (in `config.env`, default 8181) | QMD daemon, MCP config | Port for QMD HTTP daemon. Must match `settings.json` MCP config. |
| `OLLAMA_BASE_URL` | User (in `config.env`, default localhost:11434) | Ollama MCP, ingestion pipelines | Ollama server URL. |

`VAULT_ROOT` is a **path** (e.g., `/home/user/Obsidian_Work_Vault`). `OBSIDIAN_VAULT_NAME` is a **name string** (e.g., `"Obsidian_Work_Vault"`). They refer to the same vault through different interfaces — `VAULT_ROOT` for filesystem operations, `OBSIDIAN_VAULT_NAME` for Obsidian CLI operations.

```bash
export QMD_DAEMON_PORT=8181
export OLLAMA_BASE_URL="http://127.0.0.1:11434"
export OBSIDIAN_VAULT_NAME="Obsidian_Test_Vault"
```

---

## Verification

| ID | Test | Expected | Method |
|----|------|----------|--------|
| V1.1 | QMD collection exists with documents | document_count > 0 | `qmd status --json` |
| V1.2 | QMD daemon responds | HTTP 200 | `curl localhost:8181/health` |
| V1.3 | FileChanged triggers qmd update | Re-index within 35s of .md change | `touch test.md`, wait 35s, check update log |
| V1.4 | `qmd embed` works manually | Embeddings generated | Run `qmd embed`, verify vector count > 0 |
| T1.1 | BM25 search returns results | Ranked results | `qmd search "test" --collection <vault>` |
| T1.2 | MCP search tool functional | Returns MCP results | `mcp__plugin_qmd_qmd__search` |
| T1.3 | MCP get tool retrieves document | Full content returned | `mcp__plugin_qmd_qmd__get` |
| T1.4 | Daemon idempotent on double ensure | Single process | Call ensure twice, verify one PID |
| T1.5 | systemd service active + enabled | `active (running)`, `enabled` | `systemctl --user is-active/is-enabled qmd-daemon` |

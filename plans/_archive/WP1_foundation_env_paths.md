# WP1: Foundation — Environment & Path Conventions

**Status**: specified
**Severity**: HIGH
**Created**: 2026-05-13
**Implemented**: —
**Depends on**: None

---

## Problem

The `.claude` configuration contains hardcoded absolute paths (e.g., `/home/cunger/` in settings.json, hooks.json) that break when synced to other workstations. No environment variable convention exists for vault paths, QMD index directories, or API endpoints. Every script that touches the filesystem uses hardcoded strings. This blocks all subsequent phases.

---

## Verified Evidence

- `~/.claude/settings.json` — contains `/home/cunger/` path references (wrong user)
- `~/.claude/hooks/hooks.json` — multiple hardcoded `/home/cunger/` paths in hook commands
- `~/.claude/scripts/` — no `env.sh`, no `config.env`, no path resolution utilities exist
- `Obsidian_Work_Vault/System/000_Scripts/agentic_note_ingestion/.env` — vault paths hardcoded per workstation

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1 | Create `~/.claude/config.env` | pending |
| 2 | Create `~/.claude/scripts/lib/env.sh` | pending |
| 3 | Create `~/.claude/scripts/lib/vault-registry.sh` | pending |
| 4 | Create `~/.claude/scripts/lib/vault-registry.json` | pending |
| 5 | Fix `~/.claude/settings.json` paths | pending |
| 6 | Fix `~/.claude/hooks/hooks.json` paths | pending |

---

## Step-by-step

### Step 1 — config.env

User-editable file. Single source of workstation-specific configuration.

```bash
# --- Paths (edit per workstation) ---
export VAULTS_BASE="/media/${USER}/M2.SSD1/000_Vaults"
export PROJECTS_BASE="/media/${USER}/Samsung_Evo990/Projects"

# --- Vault paths ---
export OBSIDIAN_WORK_VAULT="${VAULTS_BASE}/Obsidian_Work_Vault"
export OBSIDIAN_TEST_VAULT="${VAULTS_BASE}/Obsidian_Test_Vault"
export OBSIDIAN_PERSONAL_VAULT="${VAULTS_BASE}/Obsidian_Personal_Vault"
export OBSIDIAN_ACTIVE_VAULT="${OBSIDIAN_WORK_VAULT}"

# --- QMD ---
export QMD_INDEX_DIR="${HOME}/.cache/qmd"
export QMD_DAEMON_PORT=8181

# --- Obsidian REST API (fallback) ---
export OBSIDIAN_API_URL="https://127.0.0.1:27124"
export OBSIDIAN_API_KEY=""

# --- Ollama ---
export OLLAMA_BASE_URL="http://127.0.0.1:11434"
```

**Security note on config.env:** `config.env` is gitignored in the dotfiles repo (add to `.gitignore` if not already present). It may contain API keys and workstation-specific paths. Never commit it. Provide `config.env.example` with placeholder values for documentation.

### Step 2 — env.sh

Sourced by all scripts. Provides:
- `source config.env`
- `validate_required_vars()` — checks mandatory vars are set
- `resolve_path(path_with_vars)` — expands `${VAR}` references

### Step 3 — vault-registry.sh

CLI contract:
```bash
vault list                    # {"vaults": [...], "active": "work"}
vault switch <name>           # {"previous": "work", "current": "<name>"}
vault info [--name <n>]       # {"name": "...", "path": "...", ...}
```

### Step 4 — vault-registry.json

```json
{
  "version": 1,
  "active": "work",
  "vaults": {
    "work": {
      "path": "${OBSIDIAN_WORK_VAULT}",
      "description": "Primary professional knowledge base",
      "qmd_collection": "work-vault"
    }
  }
}
```

### Step 5-6 — Fix hardcoded paths

Replace `/home/cunger/` with `~` or `${HOME}` in settings.json and hooks.json.

---

## Verification

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| Shell syntax | `bash -n ~/.claude/scripts/lib/env.sh` | No errors |
| Shell syntax | `bash -n ~/.claude/scripts/lib/vault-registry.sh` | No errors |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V0.1 | No hardcoded `/home/` or `/media/` in scripts | 0 grep hits | `grep -r '/home/\|/media/' ~/.claude/scripts/` (exclude config.env) |
| V0.2 | env.sh is idempotent | No duplicate env vars | Source twice, diff `env` output |
| V0.3 | Vault paths resolve | All paths exist | `vault list` JSON, `test -d` each path |
| T1 | config.env syntax valid | No errors | `bash -n ~/.claude/config.env` |
| T2 | vault list returns valid JSON | Valid JSON, active field present | `vault list \| jq .` |
| T3 | vault switch updates env | OBSIDIAN_ACTIVE_VAULT changes | Source env.sh, vault switch, echo var |

---

## Review

| Reviewer | Verdict | Notes |
|-----------|---------|-------|
| — | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| — | — | — |

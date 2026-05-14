# WP4: Multi-Vault Management

**Status**: specified
**Severity**: MEDIUM
**Created**: 2026-05-13
**Implemented**: —
**Depends on**: WP1, WP2

---

## Problem

Only one vault exists (`Obsidian_Work_Vault`). No mechanism to create new vaults with standard structure, register them, switch between them, or ensure QMD/mcpvault are configured for each. The user needs multiple vaults for different purposes (work, test, personal) with consistent tooling across all of them.

---

## Verified Evidence

- `${VAULTS_BASE}/` contains `Obsidian_Work_Vault/`, `test-vault/` (2 loose .md files, not a real vault), and `tmp.md`
- `test-vault/` is a pre-existing artifact (2 files: `create a link.md`, `Welcome.md`) — not a properly structured Obsidian vault. It will be REPLACED by the new `Obsidian_Test_Vault` created in Step 4.
- No vault creation scripts or templates exist in `.claude/`
- `vault-registry.sh` (WP1) provides `vault list|switch|info` but `vault create` is not yet implemented
- Work vault uses a specific folder structure: `003_Projects/`, `Areas/`, `System/`, `attachments/`, `00_raw/`

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1 | Create `~/.obsidian-vault-template/` skeleton | pending |
| 2 | Create `~/.claude/scripts/vault/create-vault.sh` | pending |
| 3 | Create `~/.claude/scripts/vault/test-vault.sh` | pending |
| 4 | Create test vault at `${VAULTS_BASE}/Obsidian_Test_Vault/` | pending |
| 5 | Register in vault-registry.json, add QMD collection, add mcpvault config | pending |
| 6 | Add `--delete` flag to create-vault.sh for vault removal | pending |

---

## Step-by-step

### Step 1 — Vault template

```
~/.obsidian-vault-template/
  .obsidian/
    app.json                  # Minimal config
    appearance.json
    community-plugins.json    # templater-obsidian only
    core-plugins.json
    plugins/
      templater-obsidian/     # Pre-installed
  .gitignore                  # .obsidian/workspace*, .smart-env/
  000_Projects/               # Empty, ready for projects
  Areas/                      # Empty knowledge areas
  System/                     # Scripts, templates
    00_Templates/
      Daily Note Template.md
  attachments/                # Media attachments
  00_raw/                     # Raw imports
  README.md                   # Vault overview
```

### Step 2 — create-vault.sh

```bash
vault create <name> [--from-template <path>]

# Procedure:
# 1. mkdir -p ${VAULTS_BASE}/Obsidian_<Name>
# 2. cp -r ~/.obsidian-vault-template/* ${VAULTS_BASE}/Obsidian_<Name>/
# 3. cd ${VAULTS_BASE}/Obsidian_<Name> && git init
# 4. Register in vault-registry.json
# 5. qmd collection add <name>-vault ${VAULTS_BASE}/Obsidian_<Name> --mask "**/*.md"
# 6. qmd update && qmd embed  (NOTE: both commands are GLOBAL — no --collection flag exists)
# 7. Add mcpvault MCP config entry

# Output contract:
{"created": true, "name": "test", "path": "/media/.../Obsidian_Test_Vault",
 "qmd_collection": "test-vault", "mcpvault_server": "mcpvault-test",
 "git_initialized": true}
```

### Step 3 — test-vault.sh

Validates vault creation:
1. Create test vault
2. Verify directory structure
3. Verify QMD collection exists and has documents (at least README.md)
4. Verify mcpvault MCP config present
5. Verify vault-registry.json updated
6. Verify git repo initialized

### Step 4-5 — Create test vault

Execute `vault create test` and verify all integration points.

### Step 6 — Vault delete

```bash
vault delete <name> [--force]

# Procedure:
# 1. Confirm vault exists in registry
# 2. Dry-run: list what will be removed (directory, registry entry, QMD collection, mcpvault config)
# 3. Require --force flag to proceed
# 4. qmd collection drop <name>-vault
# 5. Remove from vault-registry.json
# 6. Remove mcpvault MCP config entry
# 7. rm -rf ${VAULTS_BASE}/Obsidian_<Name>/

# Output contract:
{"deleted": true, "name": "test", "path_removed": "...",
 "qmd_collection_dropped": "test-vault", "registry_updated": true}
```

---

## Verification

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| Shell syntax | `bash -n ~/.claude/scripts/vault/create-vault.sh` | No errors |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V3.1 | vault create produces valid vault dir | All standard folders exist, .obsidian/ present | `test -d` each expected directory |
| V3.2 | vault switch updates all references | OBSIDIAN_ACTIVE_VAULT, QMD search scope | Switch, verify env var, verify qmd collection active |
| T1 | vault create registers in vault-registry.json | New vault appears in `vault list` | Create, list, grep |
| T2 | vault create adds QMD collection | Collection exists, has documents | `qmd status --json` after create |
| T3 | vault create initializes git | `.git/` directory present, initial commit exists | `test -d .git`, `git log` |
| T4 | vault create adds mcpvault MCP config | Config entry in mcp-servers.json | grep mcp-servers.json for new entry |
| T5 | vault create is idempotent-safe | Refuses to overwrite existing vault | Run create twice, expect error on second |
| T6 | vault switch changes QMD search scope | Search defaults to active vault's collection | Switch, search without --collection flag |
| T7 | vault can be opened in Obsidian | App recognizes new vault | Manual: File → Open Vault → select new vault dir |

---

## Review

| Reviewer | Verdict | Notes |
|-----------|---------|-------|
| — | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| Personal vault creation (deferred or now?) | LOW | Open question Q4 |
| Vault naming convention standardization | LOW | Open question Q3 |

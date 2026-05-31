# WP0: Repository Setup, Rename & Cleanup

**Status**: pending
**Severity**: HIGH
**Created**: 2026-05-14
**Depends on**: None

---

## Problem

The repo currently named `qmd-obsidian` contains stale plans for a global-plugin architecture (`~/.claude/` target). It needs to be renamed, cleaned up, and restructured for the vault-centric `claude-vault` design. The setup skill — the core onboarding experience — must be specified as the first deliverable.

---

## Implementation Steps

| Step | Action | State |
|------|--------|-------|
| 1 | Rename GitHub repo `qmd-obsidian` → `claude-vault` via `gh repo rename` | done |
| 2 | Rename local folder `qmd-obsidian/` → `claude-vault/` | done |
| 3 | Fix git remote URL to `Truncuso/claude-vault` | done |
| 4 | Archive stale plans to `plans/_archive/` | done |
| 5 | Update `.claude-plugin/plugin.json` — name, description, keywords | done |
| 6 | Write new plan files with vault-centric architecture | pending |
| 7 | Update `hooks/hooks.json` — vault-relative paths, timeouts, guards | pending |
| 8 | Update `monitors/monitors.json` — vault-relative | pending |
| 9 | Update `settings.json` — vault-relative agent paths | pending |
| 10 | Update `config/config.env.example` — vault-local config | pending |
| 11 | Write `README.md` — vault-type overview, installation, setup | pending |
| 12 | Specify the **setup skill** — interactive onboarding flow | pending |
| 13 | Git commit + push | pending |

---

## Setup Skill Specification

The setup skill is the first thing that runs after marketplace install (or is invoked manually). It is **re-runnable** for reconfiguration.

### Flow

```
1. DETECT vault state
   ├── New empty vault → full bootstrap
   ├── Existing Obsidian vault → augment with .claude/
   ├── Plain directory → ask if this should become a vault
   └── Already has .claude/ → offer reconfiguration

2. GRILL user about vault intent
   ├── "What will this vault be used for?"
   │   ├── Daily work / projects / general knowledge → working vault
   │   ├── Research / papers / experiments → research vault (WP7)
   │   └── Courses / learning / books → learning vault (WP8)
   ├── "Do you want default setup or customize?"
   └── "Which Obsidian plugins should I enable?"
       └── Defaults per vault type, user can override

3. CONFIGURE .obsidian/
   ├── Write .gitignore (workspace files, .smart-env/, config.env)
   ├── Enable core plugins by editing .obsidian/core-plugins.json:
   │   Set "templates": true, "daily-notes": true, "command-palette": true
   └── Enable community plugins by editing .obsidian/community-plugins.json:
       Register plugin IDs in "enabledPlugins" array:
       ├── All vaults: ["templater-obsidian", "dataview"]
       ├── Working: + ["calendar"] (optional, user prompt)
       └── Research: + ["citations"] (optional, user prompt)
   └── Write minimal .obsidian/app.json with new-file-location, attachment-folder

4. BOOTSTRAP .claude/
   ├── Create directory structure
   ├── Install skills matching vault type (general + type-specific)
   ├── Write CLAUDE.md with vault-type-specific instructions
   ├── Write settings.json with MCP configs
   ├── Write hooks.json and monitors.json
   └── Write config.env.example

5. INSTALL Templater templates
   └── Copy type-appropriate templates to vault's template folder

6. INDEX with QMD
   ├── qmd collection add <vault-name> <vault-path> --mask "**/*.md"
   ├── qmd update
   └── qmd embed (inform user this is expensive, can defer)

7. VERIFY
   └── Run verification checks, report any issues
```

### Output Contract

```json
{
  "setup_complete": true,
  "vault_type": "working",
  "skills_installed": ["vault-search", "obsidian-note", "daily-note", "ingest-content", "recall", "sync-sessions", "daily-journal", "project-tracker", "quick-capture", "meeting-notes"],
  "qmd_collection": "my-working-vault",
  "qmd_docs_indexed": 0,
  "plugins_configured": ["templater-obsidian", "dataview"],
  "templates_installed": 5,
  "warnings": [],
  "next_steps": ["Run qmd embed to enable vector search", "Open Obsidian to complete plugin setup"]
}
```

---

## Verification

- [ ] Repo accessible at `github.com/Truncuso/claude-vault`
- [ ] `plugin.json` has name "claude-vault"
- [ ] All stale plans in `_archive/`, fresh plans in `plans/`
- [ ] Setup skill specification complete and actionable
- [ ] Git remote points to `Truncuso/claude-vault`

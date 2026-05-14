# Handoff: QMD + Obsidian Vault Integration System

## Session Metadata

- **Created**: 2026-05-13
- **Project**: dotfiles-claude
- **Branch**: master
- **Plan**: `plans/obsidian-qmd-integration/planning-2026-05-13-full-system-setup/`

---

## Current State Summary

Full SDD plan written with 6 work packages, verification matrix (50+ tests), and architectural decisions documented. Three-layer access model (QMD for search, mcpvault for CRUD, Obsidian CLI for live ops) chosen after evaluating all options. Implementation has NOT started — this is a pure planning artifact ready for execution.

**Key architectural decisions made:**
- mcpvault (`@bitbonsai/mcpvault`) as primary CRUD path — MCP-native, no custom bridge
- LangChain/LlamaIndex for ingestion pipeline migration (user's choice)
- QMD daemon via systemd + SessionStart fallback
- Auto-update via inotify file watcher (`qmd update` — fast file scan); `qmd embed` via manual invocation or daily cron (expensive ML inference)
- All paths env-var-based, zero hardcoded absolute paths

---

## Work Completed

### Tasks Finished

- [x] Full exploration of `.claude/` directory (skills, agents, hooks, MCP configs, templates)
- [x] Full exploration of `Obsidian_Work_Vault` (plugins, templates, agentic_note_ingestion project)
- [x] Full exploration of personal-os-skills (7 skills from Artem Zhutov)
- [x] Evaluation of all Obsidian access options (REST API, mcpvault, CLI, filesystem MCP)
- [x] QMD current state audit (v0.9.0, 0 documents, 0 collections)
- [x] Architectural design: three-layer model, tool evaluation, dependency graph
- [x] OVERVIEW.md written with executive summary, WP table, execution strategy, decisions
- [x] TODO.md written with active set, 3-phase task checklist
- [x] WP1-WP6 written with problem/evidence/steps/verification per package
- [x] FINDINGS.md written — 4 detailed findings with code paths and structural analysis
- [x] OPEN_QUESTIONS.md written — 5 active questions, escalation protocol
- [x] VERIFICATION.md written — 50+ tests across all WPs with live verification commands
- [x] HANDOFF.md (this file) written

### Files Created

| File | Change | Status |
|------|--------|--------|
| `plans/obsidian-qmd-integration/planning-2026-05-13-full-system-setup/OVERVIEW.md` | New — executive summary, WP table, decisions | complete |
| `plans/obsidian-qmd-integration/planning-2026-05-13-full-system-setup/TODO.md` | New — phased task checklist | complete |
| `plans/obsidian-qmd-integration/planning-2026-05-13-full-system-setup/WP1_foundation_env_paths.md` | New — env vars, path conventions, vault registry | complete |
| `plans/obsidian-qmd-integration/planning-2026-05-13-full-system-setup/WP2_qmd_configuration.md` | New — collections, daemon, auto-embed | complete |
| `plans/obsidian-qmd-integration/planning-2026-05-13-full-system-setup/WP3_obsidian_mcp_access.md` | New — mcpvault + Obsidian CLI | complete |
| `plans/obsidian-qmd-integration/planning-2026-05-13-full-system-setup/WP4_multi_vault_management.md` | New — vault creation, switching, templates | complete |
| `plans/obsidian-qmd-integration/planning-2026-05-13-full-system-setup/WP5_skills_and_templates.md` | New — 5 skills, 2 updates, 5 slash commands | complete |
| `plans/obsidian-qmd-integration/planning-2026-05-13-full-system-setup/WP6_agentic_ingestion_langchain.md` | New — LangChain migration, model discovery, evaluator, podcast | complete |
| `plans/obsidian-qmd-integration/planning-2026-05-13-full-system-setup/OPEN_QUESTIONS.md` | New — 5 open, 3 resolved questions | complete |
| `plans/obsidian-qmd-integration/planning-2026-05-13-full-system-setup/FINDINGS.md` | New — 4 detailed findings with code paths | complete |
| `plans/obsidian-qmd-integration/planning-2026-05-13-full-system-setup/VERIFICATION.md` | New — full matrix with 50+ tests | complete |
| `plans/obsidian-qmd-integration/planning-2026-05-13-full-system-setup/HANDOFF.md` | New — this file | complete |
| `plans/we-are-want-do-glowing-badger.md` | New — initial consolidated plan (superseded by structured files) | legacy |

### Decisions Made

| Decision | Rationale |
|----------|-----------|
| Three-layer access: QMD + mcpvault + Obsidian CLI | Complementary: search (tokens), CRUD (headless), live ops (templates) |
| mcpvault over custom bridge | Zero code to maintain, MCP-native, works without Obsidian running |
| LangChain/LlamaIndex for ingestion | User's explicit choice; better ecosystem and evaluation tooling |
| systemd + SessionStart for QMD daemon | Belt-and-suspenders: persistence + fallback |
| Auto-update: `qmd update` via inotify + `qmd embed` via manual/cron | `qmd update` is fast (file scan, safe to automate); `qmd embed` is expensive (global ML inference on all docs) — split strategy |
| JSON contracts on all script outputs | Machine-parseable, testable, enables verification matrix |
| Env-var-based path system | Portability across workstations; single config file per machine |

---

## Pending Work

### Immediate Next Steps

1. Resolve 5 open questions (Q1-Q5 in OPEN_QUESTIONS.md) — Q1 about API keys is the only potentially blocking one
2. Begin WP1 implementation: create `config.env`, `env.sh`, `vault-registry.sh`, fix hardcoded paths
3. After WP1: run WP2 and WP3 in parallel (QMD configuration + mcpvault setup)
4. After WP2: WP4 (create test vault)
5. After WP2+WP3: WP5 (skills and templates) and WP6 (LangChain migration) in parallel — they have no mutual dependency

### Blockers / Open Questions

| ID | Question | Impact |
|----|----------|--------|
| Q1 | API key rotation for obsidian-local-rest-api? | Low — REST API is fallback only, not primary path |
| Q2 | Minimal plugin set for new vaults? | Medium — affects WP4 vault template design |
| Q3 | Vault naming convention? | Low — cosmetic, easy to change later |
| Q4 | Create personal vault now? | Low — can be done anytime, no dependencies on it |
| Q5 | TypeScript wrapper for ingestion CLI? | Medium — affects WP6 scope |

### Deferred Items

| Item | Reason | Priority |
|------|--------|----------|
| Systemd unit for auto-update on all vaults | Implement per-vault as vaults are created | LOW |
| Skill-creation scaffolding script | Nice-to-have for future skill development | LOW |
| CI/CD for verification matrix | Requires CI environment setup | LOW |
| Multi-machine sync testing | Only relevant when second workstation is available | LOW |

---

## Context for Resuming Agent

### Important Context

- The user has a very mature `.claude` configuration — don't rewrite existing skills/agents, extend them
- The user's communication style: voice-dictated, sometimes typo-heavy. Parse intent, don't get stuck on typos
- The `agentic_note_ingestion` project under `Obsidian_Work_Vault/System/000_Scripts/` is the user's own draft work — treat it with respect, evolve it, don't replace it
- All paths use env vars. The `config.env` file is the SINGLE source of workstation-specific config
- QMD's CLAUDE.md instructs AI agents to never run `qmd collection add`, `qmd embed`, or `qmd update` autonomously — write commands for the user to run instead. This is **agent-level policy** (the AI shouldn't silently incur compute cost or modify the index), not a system architecture constraint. User-configured automation (systemd watchers for `qmd update`, cron for `qmd embed`) is fine — the user consciously set it up. The agent's job: write example commands, not execute them. The user's job: run setup scripts and configure automation.
- mcpvault serves one vault per instance — one MCP server entry per vault
- Obsidian CLI requires the Obsidian app to be running

### Assumptions Made

- Ollama is running on `localhost:11434` (standard port)
- Node.js v20.17.0 via nvm is available for QMD and npx (mcpvault)
- Python 3.x is available for ingestion pipeline and tests
- The Obsidian desktop app can open multiple vaults (File → Open Vault)
- `inotify-tools` must be installed (`sudo apt install inotify-tools`) for the auto-update file watcher
- The user will run `setup-collections.sh` manually (per QMD's own guidelines)

### Potential Gotchas

- **inotify watch limit**: Large vaults may exceed `fs.inotify.max_user_watches` (default 8192). Document `sudo sysctl fs.inotify.max_user_watches=524288` in SETUP.md
- **QMD daemon port conflict**: If port 8181 is in use, QMD daemon fails silently. The `qmd-daemon.sh ensure` script checks before starting.
- **mcpvault npx cold start**: First invocation downloads packages, takes 10-30s. Subsequent calls use npx cache.
- **obsidian-local-rest-api SSL cert**: Self-signed cert causes verification errors. If used as fallback, Python bridge needs `verify=False`
- **smolagents → LangChain migration**: Keep existing extraction services (they're framework-agnostic). Only refactor the orchestration layer.
- **Backward compat with Templater templates**: The templates call Python via `exec`. If CLI interface changes, old templates break. Add new entrypoints alongside old ones, don't modify old ones.
- **Git-tracked `.claude/`**: The dotfiles repo at `/home/christoph/dotfiles/claude/` is symlinked to `~/.claude/claude`. Changes to skills/agents/rules/ need to be committed there.

---

## Codebase Understanding

### Architecture Overview

The `.claude/` configuration follows a layered architecture:
- **CLAUDE.md** — normative policy (single source of behavioral truth)
- **AGENTS.md** — agent catalog + orchestration routing (no policy, catalog only)
- **skills/** — 32 skill directories, each with SKILL.md frontmatter + scripts + templates
- **agents/** — 39 agent definitions (YAML frontmatter, tool allowlists, model bindings)
- **hooks/hooks.json** — 30+ lifecycle hooks across all Claude Code events
- **rules/common/** — 10 rule files encoding detailed standards per domain
- **templates/sdd/** — 7 SDD templates (WP, OVERVIEW, TODO, BUG, FINDINGS, MEMORY, OPEN_QUESTIONS)
- **mcp-configs/** — MCP server catalog (26 servers defined)

### Critical Files Discovered During Exploration

| File | Significance |
|------|-------------|
| `~/.claude/plugins/marketplaces/qmd/CLAUDE.md` | QMD's own CLAUDE.md — contains the agent-level prohibition on auto-running commands. Read this first before touching QMD config. |
| `Obsidian_Work_Vault/.obsidian/plugins/obsidian-local-rest-api/data.json` | Contains API key for REST API plugin (port 27124). Use as fallback, not primary. |
| `~/.claude/settings.json` | Has `/home/cunger/` paths (19 in hooks.json, 1 here) — WP1 must fix these. |
| `~/.cache/qmd/index.sqlite` | QMD's SQLite index — 72KB, empty. After WP2, this will grow significantly. |

### Key Patterns Discovered

- **JSON contract pattern**: All scripts output structured JSON with `status`, error contracts (`error_code`, `message`, `details`). Follow this for all new scripts.
- **Skill structure**: `SKILL.md` (YAML frontmatter + instructions) + `scripts/` + `templates/` + `workflows/`. New skills must follow this.
- **Env var resolution**: `RuntimeConfig` dataclass in `agentic_note_ingestion/src/ingestion/common/config.py` shows the pattern for env-based config. Reuse for new Python code.
- **Templater → Python bridge**: Obsidian Templater templates call Python orchestrators via `exec`/`execSync`. Keep this bridge working during migration.

---

## Related Resources

| Resource | Path / URL | Relevance |
|----------|-----------|-----------|
| QMD CLAUDE.md | `~/.claude/plugins/marketplaces/qmd/CLAUDE.md` | QMD's own agent instructions — read before touching QMD |
| QMD MCP plugin | `~/.claude/plugins/cache/qmd/qmd/0.1.0/` | QMD installation with MCP server |
| mcpvault | `https://github.com/bitbonsai/mcpvault` | Primary CRUD MCP server for Obsidian vaults |
| Obsidian CLI docs | `https://obsidian.md/cli` | Obsidian CLI reference (search, create, daily, eval) |
| personal-os-skills | `/media/christoph/Samsung_Evo990/Projects/00_AI/00_Workflows/personal-os-skills/` | Source for `recall` and `sync-claude-sessions` skills |
| agentic_note_ingestion | `Obsidian_Work_Vault/System/000_Scripts/agentic_note_ingestion/` | Ingestion project to be migrated to LangChain |
| Obsidian Work Vault | `Obsidian_Work_Vault/.obsidian/plugins/` | 26 installed plugins including obsidian-local-rest-api |

---

## Environment State

- **Build**: N/A (no build step for shell scripts)
- **QMD**: v0.9.0 installed, 0 documents indexed
- **Git**: master branch, clean (only `plugins/known_marketplaces.json` modified)
- **Uncommitted changes**: `plugins/known_marketplaces.json` modified; new plan directory untracked
- **Vaults**: 1 existing (Obsidian_Work_Vault), 0 additional
- **Skills**: 32 existing, 0 new created yet
- **MCP servers**: 26 defined, 0 Obsidian-related, QMD HTTP not yet configured

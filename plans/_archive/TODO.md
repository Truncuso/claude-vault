# TODO.md — QMD + Obsidian Vault Integration System

**Date**: 2026-05-13
**Updated**: 2026-05-13
**Status**: specified

---

## Active Set

| WP | Title | Severity | Status / Next Action |
|----|-------|----------|----------------------|
| WP1 | Foundation — Environment & Path Conventions | HIGH | specified — ready for implementation |
| WP2 | QMD Configuration — Collections, Indexing, Daemon, Auto-Update | HIGH | specified — depends on WP1 |
| WP3 | Obsidian MCP Access — mcpvault + CLI | HIGH | specified — depends on WP1 |
| WP4 | Multi-Vault Management | MEDIUM | specified — depends on WP1, WP2 |
| WP5 | Skills & Templates | MEDIUM | specified — depends on WP2, WP3 |
| WP6 | Agentic Ingestion — LangChain/LlamaIndex Migration | HIGH | specified — depends on WP2, WP3 (parallel with WP5) |

---

## Phase 1: Planning & Research

- [x] Explore existing `.claude/` directory structure, skills, agents, hooks, MCP configs
- [x] Explore existing vaults (`Obsidian_Work_Vault`), agentic_note_ingestion project, templates
- [x] Explore personal-os-skills (Artem Zhutov's 7 skills including recall, sync-claude-sessions)
- [x] Evaluate Obsidian access options: obsidian-local-rest-api, mcpvault, Obsidian CLI, filesystem MCP
- [x] Evaluate QMD current state (v0.9.0, 0 documents, 0 collections)
- [x] Architectural decisions: three-layer model, mcpvault over bridge, LangChain migration
- [x] Write OVERVIEW.md, FINDINGS.md, WP1-WP6, VERIFICATION.md, TODO.md, OPEN_QUESTIONS.md

### Open Questions (block implementation)

| ID | Question | Blocks |
|----|----------|--------|
| Q1 | API key rotation for obsidian-local-rest-api? | WP3 (if we use REST API as fallback) |
| Q2 | Minimal plugin set for new vault templates? | WP4 |
| Q3 | Vault naming convention: `Obsidian_<Purpose>_Vault` or `Obsidian_<Purpose>`? | WP4 |
| Q4 | Create personal vault now or defer? | WP4 |
| Q5 | TypeScript CLI wrapper for ingestion, or keep Python-only? | WP6 |

---

## Phase 2: Implementation

### WP1 — Foundation
- [ ] Install system dependencies: `apt install inotify-tools` (for WP2 auto-update watcher)
- [ ] Create `~/.claude/config.env` with env var definitions
- [ ] Create `~/.claude/scripts/lib/env.sh` — sources config.env, validates vars, `resolve_path()`
- [ ] Create `~/.claude/scripts/lib/vault-registry.sh` — `vault list|switch|create|info` CLI
- [ ] Create `~/.claude/scripts/lib/vault-registry.json` — machine-readable registry
- [ ] Fix hardcoded `/home/cunger/` paths in `~/.claude/settings.json`
- [ ] Fix hardcoded paths in `~/.claude/hooks/hooks.json`

### WP2 — QMD Configuration
- [ ] Create `~/.claude/scripts/qmd/setup-collections.sh` — idempotent collection setup (manual run)
- [ ] Create `~/.claude/scripts/qmd/qmd-daemon.sh` — start/stop/status/ensure
- [ ] Create `~/.claude/scripts/qmd/auto-update.sh` — inotify-based watcher → `qmd update` (fast file scan)
- [ ] Create `~/.config/systemd/user/qmd-daemon.service`
- [ ] Create `~/.config/systemd/user/qmd-auto-update.service`
- [ ] Document `qmd embed` as manual/cron operation (daily cron recommended)
- [ ] Create context description files for QMD sub-path contexts
- [ ] Add SessionStart hook for QMD daemon ensure
- [ ] Add QMD HTTP MCP config to `mcp-servers.json`

### WP3 — Obsidian MCP Access
- [ ] Add mcpvault MCP config per vault to `mcp-servers.json`
- [ ] Create `~/.claude/scripts/obsidian/obsidian-cli.sh` — wrapper with JSON output
- [ ] Test mcpvault CRUD roundtrip

### WP4 — Multi-Vault Management
- [ ] Create `~/.obsidian-vault-template/` skeleton directory
- [ ] Create `~/.claude/scripts/vault/create-vault.sh`
- [ ] Create `~/.claude/scripts/vault/test-vault.sh`
- [ ] Create test vault at `${VAULTS_BASE}/Obsidian_Test_Vault/`
- [ ] Register in vault-registry.json, add QMD collection, add mcpvault config

### WP5 — Skills & Templates
- [ ] Copy `recall/` and `sync-claude-sessions/` from personal-os-skills into `~/.claude/skills/`
- [ ] Create `skills/vault-manager/` (SKILL.md + scripts/)
- [ ] Create `skills/vault-search/` (SKILL.md + workflows/) — include search routing table (QMD default, mcpvault fallback)
- [ ] Create `skills/obsidian-note/` (SKILL.md + templates/)
- [ ] Create `skills/daily-note/` (SKILL.md)
- [ ] Create `skills/ingest-content/` (SKILL.md + workflows/)
- [ ] Update `skills/recall/` — fix paths, add vector search, collection scoping
- [ ] Update `skills/sync-claude-sessions/` — fix paths, QMD re-index trigger
- [ ] Create 5 slash commands in `commands/`

### WP6 — Agentic Ingestion
- [ ] Create `src/ingestion/common/model_discovery.py` — Ollama + API model enumeration
- [ ] Create `src/ingestion/common/prompt_registry.py` — versioned YAML/JSON prompts
- [ ] Create `src/ingestion/common/source_adapter.py` — unified ContentSource protocol
- [ ] Create `src/ingestion/common/evaluator.py` — output quality scoring
- [ ] Create `src/ingestion/podcast/` — podcast ingestion pipeline
- [ ] Refactor `src/ingestion/youtube/` — smolagents → LangChain
- [ ] Refactor `src/ingestion/pdf/` — smolagents → LangChain
- [ ] Create `src/ingestion/cli/main.py` — unified CLI entrypoint
- [ ] Create `prompts/` — versioned prompt templates (youtube_summary, pdf_extraction, podcast_notes)
- [ ] Update tests for LangChain migration
- [ ] Update ARCHITECTURE.md and SETUP.md

---

## Phase 3: Verification & Review

- [ ] Run V0.1: grep for hardcoded paths (BLOCK)
- [ ] Run V0.2: env.sh idempotency (BLOCK)
- [ ] Run V0.3: vault paths resolve (BLOCK)
- [ ] Run V1.1-V1.4: QMD collections, daemon, auto-update, embed (BLOCK/WARN)
- [ ] Run V2.1-V2.3: mcpvault tools, CRUD, CLI search (BLOCK/WARN)
- [ ] Run V3.1-V3.2: vault create, switch (BLOCK)
- [ ] Run V4.1-V4.2: skills load, vault-search E2E (BLOCK/WARN)
- [ ] Run V5.1-V5.4: ingestion contracts, model discovery, evaluator, backward compat (BLOCK/WARN)
- [ ] Run V6.1-V6.3: unit, integration, E2E test suites (BLOCK/WARN)
- [ ] Run code-reviewer agent on all changes
- [ ] Run security-reviewer on MCP configs
- [ ] Archive completed WPs

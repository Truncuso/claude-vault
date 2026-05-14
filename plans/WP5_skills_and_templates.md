# WP5: Skills & Templates

**Status**: specified
**Severity**: MEDIUM
**Created**: 2026-05-13
**Implemented**: —
**Depends on**: WP2, WP3

---

## Problem

No skills exist for vault management, semantic search across vaults, note creation in Obsidian, daily note operations, or unified content ingestion. Existing skills from personal-os-skills (recall, sync-claude-sessions) have hardcoded paths and don't use the env var system. Skills are the primary user interface for all vault operations — without them, the infrastructure from WP1-WP4 is inaccessible.

---

## Verified Evidence

- `.claude/skills/` — 32 skills, none related to vault management, search, or Obsidian operations
- personal-os-skills `recall` skill — uses QMD BM25, has hardcoded paths, no vector search mode, no `--collection` scoping
- personal-os-skills `sync-claude-sessions` skill — 31KB Python script, hardcoded paths, no QMD re-index trigger
- `.claude/commands/` — 14 slash commands, none for vault operations

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 0 | Copy `recall/` and `sync-claude-sessions/` from personal-os-skills into `~/.claude/skills/` | pending |
| 1 | Create `skills/vault-manager/` (SKILL.md + scripts/) | pending |
| 2 | Create `skills/vault-search/` (SKILL.md + workflows/) | pending |
| 3 | Create `skills/obsidian-note/` (SKILL.md + templates/) | pending |
| 4 | Create `skills/daily-note/` (SKILL.md) | pending |
| 5 | Create `skills/ingest-content/` (SKILL.md + workflows/) | pending |
| 6 | Update `skills/recall/` — fix paths, add vector search, collection scoping | pending |
| 7 | Update `skills/sync-claude-sessions/` — fix paths, QMD re-index trigger | pending |
| 8 | Create 5 slash commands in `commands/` | pending |

---

## Step-by-step

### Step 0 — Install skills from personal-os-skills

`recall` and `sync-claude-sessions` exist only in the personal-os-skills repo at `/media/christoph/Samsung_Evo990/Projects/00_AI/00_Workflows/personal-os-skills/skills/`. They must be copied into `~/.claude/skills/` before they can be modified:

```bash
cp -r /media/christoph/Samsung_Evo990/Projects/00_AI/00_Workflows/personal-os-skills/skills/recall \
     ~/.claude/skills/recall
cp -r /media/christoph/Samsung_Evo990/Projects/00_AI/00_Workflows/personal-os-skills/skills/sync-claude-sessions \
     ~/.claude/skills/sync-claude-sessions
```

Steps 6-7 then modify these installed copies to use env vars and add QMD integration.

### Skill structure convention

All new skills follow the established pattern with additions:

```
skills/<name>/
  SKILL.md              # YAML frontmatter (name, description, allowed-tools, argument-hint) + instructions
  scripts/              # Shell/Python scripts specific to this skill
  templates/            # Note templates for Obsidian output
  workflows/            # LLM workflow definitions (prompt chains, routing logic)
```

### Skill 1: vault-manager

```yaml
name: vault-manager
description: Multi-vault operations — create, switch, list, validate vaults. Routes to vault.sh for execution.
allowed-tools: Bash(vault:*)
triggers: "manage vaults", "create vault", "switch vault", "list vaults"
```

### Skill 2: vault-search

```yaml
name: vault-search
description: Semantic search across Obsidian vaults via QMD. Combines BM25, vector search, and LLM synthesis.
allowed-tools: Bash(qmd:*), mcp__plugin_qmd_qmd:*
argument-hint: "[QUERY]"
triggers: "search vault", "find in notes", "what do I have about", "search my knowledge base"

# Modes:
# 1. Quick — BM25 keyword search, returns ranked file list
# 2. Deep — Query expansion + vector + BM25 fusion with synthesized answer
# 3. Targeted — Get specific document by path or docid
```

Workflow (`workflows/search.md`): routing logic for quick vs. deep vs. targeted search based on query complexity.

**Search tool routing table** (resolves three-layer redundancy — QMD, mcpvault, and CLI all offer search):

| Condition | Tool | Reason |
|-----------|------|--------|
| Default (all queries) | QMD `search` / `deep_search` | Indexed BM25 + optional vector. Fastest, richest results. |
| QMD daemon down | mcpvault `search_notes` | Fallback — filesystem-level BM25, works without QMD |
| Need current open file content | Obsidian CLI `read` / `eval` | Only Obsidian knows which file is currently focused |
| Need to search within active note | Obsidian CLI `eval "editor.getDoc().getValue()"` | In-editor content not yet saved to disk |


### Skill 3: obsidian-note

```yaml
name: obsidian-note
description: Create, read, update notes in the active Obsidian vault via mcpvault.
allowed-tools: mcp__mcpvault-work:*, Bash(obsidian-cli:*)
triggers: "create note", "write to obsidian", "update note", "read note from vault"
```

Templates (`templates/`): Meeting Note, Project Overview, Research Summary, Daily Log.

### Skill 4: daily-note

```yaml
name: daily-note
description: Create or append to the daily note in the active Obsidian vault.
allowed-tools: Bash(obsidian:*)
triggers: "daily note", "journal entry", "log this"
```

### Skill 5: ingest-content

```yaml
name: ingest-content
description: Unified ingestion entrypoint. Routes YouTube/PDF/podcast/URL to appropriate LangChain pipeline.
allowed-tools: Bash(python:*), Bash(qmd:*)
argument-hint: "[URL or FILE]"
triggers: "ingest this", "process this video", "summarize this PDF", "import this podcast"
```

Workflow (`workflows/routing.md`): source type detection → model selection → pipeline execution → vault write → QMD re-index.

### Skill updates: recall

- Replace all hardcoded paths with `${VAULTS_BASE}`, `${HOME}/.claude/sessions`
- Add `--mode vector` for semantic (vector) search alongside existing BM25
- Add `--collection <name>` for scoped search within a specific vault
- Add QMD collection context injection for relevance boosting

### Skill updates: sync-claude-sessions

- Replace hardcoded vault paths with `${OBSIDIAN_ACTIVE_VAULT}`
- Add post-export QMD re-index: `qmd update && qmd embed` (both commands are global — no --collection flag exists)
- Add frontmatter validation against schema (existing `schema/tags.yaml`)

### Slash commands

| Command | Maps to | Purpose |
|---------|---------|---------|
| `/vault-manager` | vault-manager skill | Multi-vault operations |
| `/vault-search` | vault-search skill | Semantic search |
| `/obsidian-note` | obsidian-note skill | Note CRUD |
| `/daily-note` | daily-note skill | Daily journal |
| `/ingest` | ingest-content skill | Content ingestion |

---

## Verification

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V4.1 | All 5 new skills load without error | SKILL.md frontmatter parsed, description displayed | Skill invocation |
| V4.2 | vault-search returns synthesized results | QMD search + LLM synthesis from vault content | E2E with known query |
| T1 | vault-manager routes to vault.sh correctly | `vault list` executed, output parsed | Invoke skill, verify subcommand |
| T2 | vault-search quick mode returns file list | Ranked results from QMD BM25 | Search for known content |
| T3 | vault-search deep mode synthesizes answer | Multiple QMD calls + LLM combination | Complex query requiring multi-document synthesis |
| T4 | obsidian-note creates file via mcpvault | File visible in vault, content correct | Create note, read back |
| T5 | daily-note appends to existing daily note | Content appended, not overwritten | Append twice, verify both entries |
| T6 | ingest-content routes YouTube correctly | YouTube pipeline called | Provide YT URL, verify orchestrator invoked |
| T7 | ingest-content routes PDF correctly | PDF pipeline called | Provide PDF path, verify orchestrator invoked |
| T8 | recall works with env vars (no hardcoded paths) | Search succeeds, paths from env | Invoke recall after WP1 |
| T9 | sync-claude-sessions triggers QMD re-index | sessions collection updated after export | Export session, check qmd status |
| T10 | All 5 slash commands registered | Commands appear in help/list | `/help` or check commands/ directory |

---

## Review

| Reviewer | Verdict | Notes |
|-----------|---------|-------|
| — | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| skill-creation script to scaffold new skills from template | LOW | Future enhancement |

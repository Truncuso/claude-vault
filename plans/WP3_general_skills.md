# WP3: General Skills (All Vault Types)

**Status**: pending
**Severity**: HIGH
**Created**: 2026-05-14
**Depends on**: WP2

---

## Problem

A vault with QMD + mcpvault + Ollama MCP has powerful tools but no AI skill layer. Six core skills are needed for every vault regardless of type: search, CRUD, daily notes, content ingestion, cross-session recall, and session export.

---

## Implementation Steps

| Step | Skill | State |
|------|-------|-------|
| 1 | `vault-search` — Three-mode QMD search with LLM synthesis | pending |
| 2 | `obsidian-note` — CRUD via mcpvault | pending |
| 3 | `daily-note` — Daily journal operations via Obsidian CLI | pending |
| 4 | `ingest-content` — Unified routing to LangChain pipelines | pending |
| 5 | `recall` — Cross-session memory retrieval | pending |
| 6 | `sync-sessions` — Export sessions to vault notes + QMD re-index | pending |

---

## Skill Specifications

### 1. vault-search

Three modes mapping to QMD search modes:

| Mode | QMD Tool | Description |
|------|----------|-------------|
| `quick` | `search` (BM25) | Fast keyword search, returns file list with excerpts |
| `standard` | `deep_search` | Auto-expands query, hybrid BM25 + vector, reranked |
| `deep` | Multiple `deep_search` + LLM | Multi-query synthesis across documents |

**Search routing (fallback chain):**
```
QMD search → if down/empty: mcpvault search_notes → if Obsidian running: obsidian-cli search
```

**SKILL.md frontmatter:**
```yaml
name: vault-search
description: "Search vault with QMD semantic search. Three modes: quick, standard, deep. Auto-falls back to mcpvault/Obsidian CLI if QMD unavailable."
argument-hint: "[query] [--mode quick|standard|deep] [--vault-scope <path>]"
```

### 2. obsidian-note

CRUD operations via mcpvault. Validates frontmatter, checks wikilinks.

```yaml
name: obsidian-note
description: "Create, read, update, delete vault notes via mcpvault. Frontmatter-aware. Wikilink validation."
argument-hint: "[action: create|read|update|delete] [path] [--frontmatter key=value]"
```

### 3. daily-note

Daily journal operations via Obsidian CLI (requires Obsidian running).

```yaml
name: daily-note
description: "Open today's daily note, append content, read past daily notes. Uses Obsidian CLI (requires Obsidian running)."
argument-hint: "[action: open|append|read|yesterday] [content]"
```

### 4. ingest-content

Unified routing to LangChain pipelines. Detects content type from URL/extension.

```yaml
name: ingest-content
description: "Ingest YouTube videos, PDFs, podcasts, or web pages into structured vault notes. Auto-detects content type, routes to appropriate LangChain pipeline."
argument-hint: "[url-or-path] [--type youtube|pdf|podcast|web] [--style <profile>]"
```

### 5. recall

Cross-session memory from personal-os-skills. Adapted to use QMD instead of grep.

- **Temporal:** `/recall yesterday` — reconstructs sessions from a given day
- **Topic:** BM25 + vector search across QMD sessions collection
- **Graph:** Interactive visualization of sessions as colored nodes

```yaml
name: recall
description: "Cross-session memory retrieval. Temporal, topic, and graph modes. Uses QMD for semantic search over past sessions."
argument-hint: "[query] [--mode temporal|topic|graph] [--date <YYYY-MM-DD>]"
```

### 6. sync-sessions

Exports Claude Code sessions to vault markdown files. Triggers QMD re-index after export.

```yaml
name: sync-sessions
description: "Export Claude Code sessions to vault markdown. Validates frontmatter. Triggers QMD re-index after export."
argument-hint: "[--session-id <id>] [--all]"
```

**Post-export:** `qmd update && qmd embed` (global, re-indexes all collections including sessions).

---

## Skill Contracts

Every skill must specify in its SKILL.md:

1. **Error contract**: JSON output on failure — `{"error": "...", "error_code": "..."}`. Non-zero exit for scripted use.
2. **Idempotency**: Is the operation safe to repeat? `obsidian-note create` fails on duplicate path; `daily-note append` always succeeds (append-only).
3. **Side effects**: Does the operation modify filesystem only, or also trigger QMD re-index? sync-sessions explicitly triggers re-index; daily-note and obsidian-note do NOT (FileChanged hook catches their fs writes).
4. **Timeout expectation**: Approximate duration. QMD search: <2s; deep_search: <10s; LLM synthesis: 30-120s.
5. **Fallback behavior**: What happens when primary tool is unavailable? vault-search: QMD → mcpvault → Obsidian CLI fallback chain documented.

## Skill Structure Convention

```
skills/general/<name>/
├── SKILL.md              # YAML frontmatter + numbered workflow + contract section
├── references/           # Supplementary docs (examples, limitations, fallback chains)
├── templates/            # Note templates used by the skill
└── workflows/            # LLM workflow definitions (routing, chains)
```

---

## Verification

| ID | Test | Expected | Method |
|----|------|----------|--------|
| V3.1 | All 6 skills load without error | SKILL.md parsed, description displayed | Skill invocation |
| V3.2 | vault-search returns synthesized results | QMD search + LLM synthesis | E2E with known query |
| T3.1 | vault-search quick mode returns file list | BM25 ranked results | Search for known content |
| T3.2 | vault-search deep mode synthesizes multi-doc answer | Combined results from multiple QMD calls | Complex cross-topic query |
| T3.3 | obsidian-note creates file via mcpvault | Note visible, content correct | Create, read back |
| T3.4 | daily-note appends without overwriting | Both entries present | Append twice, verify concatenation |
| T3.5 | ingest-content routes YouTube correctly | YouTube pipeline invoked | Provide YT URL, verify pipeline log |
| T3.6 | ingest-content routes PDF correctly | PDF pipeline invoked | Provide PDF path, verify pipeline log |
| T3.7 | recall works with vault-relative paths | Search succeeds, no hardcoded path errors | Invoke recall |
| T3.8 | sync-sessions triggers QMD re-index | sessions collection updated | Export session, check qmd status |
| T3.9 | All slash commands registered | Commands appear in help | `/help` |

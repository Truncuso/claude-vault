# QMD + Obsidian Vault Integration System

**Date**: 2026-05-13
**Updated**: 2026-05-13
**Author**: Christoph Unger
**Status**: specified

---

## Executive Summary

Integrate QMD (local markdown semantic search engine) with Obsidian vault management into Claude Code. The user has a mature `.claude` configuration (32 skills, 42 agents, SDD workflow) but QMD has 0 indexed documents and no Obsidian MCP access exists. This plan delivers a portable, env-var-driven three-layer access model (QMD for search, mcpvault for CRUD, Obsidian CLI for live ops), manages multiple vaults, evolves the draft agentic note ingestion project from smolagents to LangChain/LlamaIndex, and enforces verifiable contracts at every layer.

**Key insight:** Three-layer access model — QMD handles all semantic/keyword search (saves tokens by finding context before loading files), mcpvault provides MCP-native CRUD without needing Obsidian running, and Obsidian CLI handles template-based note creation when the app IS running.

---

## Active Work Packages

| WP | Title | Severity | Status | Impact |
|----|-------|----------|--------|--------|
| WP1 | Foundation — Environment & Path Conventions | HIGH | specified | All files — fixes hardcoded paths, establishes env var system |
| WP2 | QMD Configuration — Collections, Indexing, Daemon, Auto-Update | HIGH | specified | New: 4 scripts, 2 systemd units, 1 hook, 1 MCP config |
| WP3 | Obsidian MCP Access — mcpvault + CLI | HIGH | specified | New: MCP configs per vault, CLI wrapper script |
| WP4 | Multi-Vault Management | MEDIUM | specified | New: 2 scripts, template dir, test vault creation |
| WP5 | Skills & Templates | MEDIUM | specified | New: 5 skills, 2 skill modifications |
| WP6 | Agentic Ingestion — LangChain/LlamaIndex Migration | HIGH | specified | Refactor: smolagents→LangChain, +podcast pipeline, +evaluator |

---

## Archived / Closed / Deleted

| WP | Outcome |
|----|---------|
| — | — |

---

## Corrections Log

| Previous Claim | Corrected Finding |
|----------------|-------------------|
| Build custom Python MCP bridge for obsidian-local-rest-api | Use mcpvault (bitbonsai/mcpvault) — MCP-native, no bridge needed, 14 tools |
| Keep smolagents framework | Migrate to LangChain/LlamaIndex per user's explicit choice |
| Auto-embed: both `qmd update` and `qmd embed` via inotify | Split: `qmd update` via inotify (fast file scan), `qmd embed` via manual/cron (expensive ML). Both commands are global — no `--collection` flag exists. |
| WP6 depends on WP5 | WP5 and WP6 are independent — both depend on WP2+WP3, no mutual dependency |

---

## Execution Strategy

```
WP1 (Foundation)
  |
  +---> WP2 (QMD) -----> WP4 (Multi-Vault)
  |        |                 |
  +---> WP3 (mcpvault+CLI) --+
           |
           +---> WP5 (Skills)
           |
           +---> WP6 (Ingestion)
```

WP2 and WP3 are parallelizable (no mutual dependencies). WP4 depends on WP2 (vault creation triggers QMD collection setup). WP5 depends on WP2+WP3 (skills use QMD and mcpvault tools). WP6 depends on WP2+WP3 (LangChain pipelines use QMD for search, mcpvault for writing output). WP5 and WP6 are independent — they can run in parallel.

---

## Key Decisions

| Decision | Rationale | Date |
|----------|-----------|------|
| Three-layer access: QMD + mcpvault + Obsidian CLI | QMD = search (token-saving), mcpvault = CRUD (MCP-native, no Obsidian needed), CLI = live ops (templates, UI). Complementary, not redundant. | 2026-05-13 |
| mcpvault over obsidian-local-rest-api bridge | mcpvault is MCP-native (zero bridge code), works without Obsidian running, 14 tools. REST API plugin stays as fallback. | 2026-05-13 |
| QMD daemon: systemd + SessionStart hook | systemd for persistence across reboots; SessionStart hook as belt-and-suspenders fallback | 2026-05-13 |
| Auto-update split strategy: `qmd update` via inotify + `qmd embed` via manual/cron | `qmd update` is fast (file scan, safe to automate). `qmd embed` is expensive (global ML inference on ALL docs) — manual or daily cron. Both commands are global (no `--collection` flag exists). | 2026-05-14 |
| LangChain/LlamaIndex for ingestion | User's explicit choice. Better ecosystem compatibility, structured output, evaluation tooling. | 2026-05-13 |
| Env-var-based paths, no hardcoded absolute paths | Enables portability across workstations. `config.env` is the single source of workstation-specific config. | 2026-05-13 |

---

## Reference Sources

All resources, documentation, repos, and blog posts referenced during planning. These are the canonical external sources for this project. Each entry includes exploration notes.

---

### Core Tools (Primary Dependencies)

| Source | URL | Role | Notes |
|--------|-----|------|-------|
| **QMD** | [github.com/tobi/qmd](https://github.com/tobi/qmd) | Local markdown search engine (BM25 FTS5 + vector via sqlite-vec + node-llama-cpp). Built by Tobias Lutke (Shopify CEO). | v0.9.0. Three search modes: `search` (~30ms BM25), `vector_search` (~2s semantic), `deep_search` (~10s hybrid). `qmd update` and `qmd embed` are GLOBAL — no per-collection scoping. |
| **QMD CLAUDE.md** | `~/.claude/plugins/marketplaces/qmd/CLAUDE.md` | QMD's own agent instructions. | Agent-level prohibition: AI must never auto-run `qmd collection add`, `qmd embed`, or `qmd update`. User-configured systemd/cron/inotify watchers are fine — the user consciously set them up. |
| **mcpvault** | [github.com/bitbonsai/mcpvault](https://github.com/bitbonsai/mcpvault) | MCP-native Obsidian vault CRUD via `npx @bitbonsai/mcpvault`. 14 tools: read/write/patch/delete/move/search notes, frontmatter, tags, directory listing. Headless (no Obsidian needed). | One vault per instance — multi-vault requires multiple MCP server entries. Token-optimized output. |
| **mcp-obsidian** (evaluated, not chosen) | [github.com/MarkusPfundstein/mcp-obsidian](https://github.com/MarkusPfundstein/mcp-obsidian) | Alternative MCP server for Obsidian. | Evaluated but rejected — mcpvault is MCP-native, requires no bridge code, and works without Obsidian running. |
| **Obsidian CLI** | [obsidian.md/cli](https://obsidian.md/cli) | CLI for running Obsidian instance. Subcommands: `open`, `search`, `create`, `daily`, `daily:append`, `read`, `eval`. | Requires Obsidian app running. Benchmark: 54x faster than grep for orphan notes, 6x faster for vault search. Output via `--json` flag. |

---

### Claude Code Plugin Documentation

| Source | URL | Role | Notes |
|--------|-----|------|-------|
| **Claude Code Plugins** | [code.claude.com/docs/en/plugins](https://code.claude.com/docs/en/plugins) | Plugin system reference: `.claude-plugin/plugin.json` manifest, `skills/`, `hooks/`, `monitors/`, `bin/`, `settings.json`. | Plugin name becomes namespace for slash commands (e.g., `/qmd-obsidian:search`). |
| **Hooks Guide** | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide) | Lifecycle hooks: SessionStart, FileChanged, PostCompact, Stop, SessionEnd. | Key pattern: SessionStart for daemon ensure + vault injection, FileChanged for async qmd update on .md changes. |
| **Monitoring Usage** | [code.claude.com/docs/en/monitoring-usage](https://code.claude.com/docs/en/monitoring-usage) | Background monitors for health checks and file watching. | Persistent monitors defined in `monitors/monitors.json`. |

---

### Blog Posts & Articles

| Source | URL | Author/Date | Key Insights |
|--------|-----|-------------|--------------|
| **"Grep Is Dead: How I Made Claude Code Actually Remember Things"** | [artemxtech.substack.com](https://artemxtech.substack.com/p/grep-is-dead-how-i-made-claude-code) | Artem Zhutov, 2026-03-02 | Core problem: 700 sessions in 3 weeks, zero cross-session memory. Solution stack: QMD + `/recall` skill + Obsidian vault. Three recall modes: temporal (reconstruct sessions from a day), topic (BM25 across collections), graph (visualization). Key metric: semantic search finds 4/5 relevant results that don't contain the search words. Philosophy: "Tools change. If you have your context you can make it work in any situation." |
| **"Obsidian + Claude Code Integration Guide"** | [blog.starmorph.com](https://blog.starmorph.com/blog/obsidian-claude-code-integration-guide) | Kevin Lee / StarMorph | Five integration strategies ranked. QMD + session sync stack achieves "60%+ token reduction vs grep/glob." Obsidian CLI v1.12: 54x faster for orphan notes. Core principle: "Agents read, humans write" — AI output in `~/.claude/`, authentic thinking in vault. |
| **"Claude Code Doesn't Index Your Codebase"** | [vadim.blog](https://vadim.blog/claude-code-no-indexing) | Vadim, 2026-03 | Confirms Claude Code uses Glob → Grep → Read → Explore agent hierarchy (NOT indexing). Boris Cherny (Claude Code creator): early RAG versions abandoned — "agentic search outperformed it by a lot." 92% prefix caching reuse. QMD fills the indexing gap that Claude Code intentionally left open. |

---

### Reference Repos (Explored & Analyzed)

#### claude-obsidian (Truncuso/claude-obsidian)

| | |
|---|---|
| **Path** | `/media/christoph/Samsung_Evo990/Projects/00_AI/00_Workflows/claude-obsidian/` |
| **URL** | [github.com/Truncuso/claude-obsidian](https://github.com/Truncuso/claude-obsidian) |
| **What it does** | Claude Code plugin building persistent, compounding Obsidian wiki vaults — drop sources, AI extracts entities/concepts, cross-references everything. |
| **Skills** (11 total) | `wiki` (orchestrator), `wiki-ingest` (8-15 pages per source), `wiki-query` (3 depths), `wiki-lint` (10-category health check), `wiki-fold` (log rollup), `save` (conversation → wiki note), `autoresearch` (3-round web loop), `canvas` (.canvas JSON), `defuddle` (strip web clutter), `obsidian-markdown` (reference), `obsidian-bases` (reference) |

**Key patterns to adapt for qmd-obsidian:**

| Pattern | Description | WP Relevance |
|---------|-------------|--------------|
| **Hot cache** (`wiki/hot.md`) | ~500-word summary of recent context, rewritten on every operation. SessionStart restores it silently, PostCompact re-loads after compaction. Solves "what were we doing?" across sessions. | WP5 (vault-search skill) — warm context injection on session start |
| **Three-layer content architecture** | `.raw/` (immutable sources) + `wiki/` (AI-generated) + schema files (CLAUDE.md, WIKI.md) as instruction layer. Clean separation of human vs. AI content. | WP6 (ingestion pipeline output) — separate raw extracts from structured notes |
| **Dry-run/commit split** | Destructive ops default to stdout dry-run. Commit mode requires explicit user confirmation. Prevents auto-commit hooks from firing on experimental output. | WP4 (vault delete — dry-run first, commit with --force) |
| **Delta tracking via manifest** | `.raw/.manifest.json` stores source hashes, prevents re-ingestion of unchanged sources. | WP6 (source adapter — skip already-ingested content) |
| **Feature detection for extensions** | Runtime checks for script/file presence with graceful fallback. No hard dependencies. | WP3 (check Obsidian CLI availability, fall back to mcpvault) |
| **Exit-code protocol** | Scripts use distinct exit codes (0=ok, 2=usage, 3=cache corrupt, 10=service unreachable, 11=model missing). Skills read exit codes explicitly. | WP6 (CLI error handling, JSON contracts) |
| **Deterministic fold IDs** | Filenames from input ranges (not timestamps) — structural idempotency. Same inputs → same filename → duplicate detection. | WP6 (ingestion idempotency) |

---

#### avoid-ai-writing (conorbronsdon/avoid-ai-writing)

| | |
|---|---|
| **Path** | `/media/christoph/Samsung_Evo990/Projects/00_AI/00_Workflows/avoid-ai-writing/` |
| **URL** | [github.com/conorbronsdon/avoid-ai-writing](https://github.com/conorbronsdon/avoid-ai-writing) |
| **What it does** | Portable single-file AI writing audit-and-rewrite skill (v3.3.1). Two modes: rewrite (audit + remove AI-isms) and detect (flag only). |

**Key patterns:**

| Pattern | Description | WP Relevance |
|---------|-------------|--------------|
| **36 pattern categories across 5 groups** | Content (significance inflation, notability name-dropping), Language (109-entry word table, 3 tiers, adapted from brandonwise/humanizer), Structure (em dashes, uniform paragraph length), Communication (chatbot artifacts, sycophantic tone), Meta (rhythm/uniformity as #1 signal per Pangram Labs research). | WP6 (evaluator — add AI-writing detection as quality dimension) |
| **6 context profiles with tolerance matrix** | `blog`, `technical-blog`, `linkedin`, `docs`, `casual`, `default`. Per-rule tolerance: strict/relaxed/skip/extra-strict. | WP6 (prompt registry — context-aware output quality) |
| **Rewrite-vs-patch threshold** | Quantitative gate: 5+ vocabulary flags + 3+ pattern categories + uniform rhythm = full rewrite. Otherwise surgical edit. | WP6 (evaluator — structured pass/fail/needs_review decisions) |
| **Integration idea** | Run ingested AI-generated notes through detect mode as post-processing gate. Flag AI-isms before they enter vault. Tag notes for review. | WP6 (post-generation quality filter) |

---

#### obsidian-skills (kepano/obsidian-skills)

| | |
|---|---|
| **Path** | `/media/christoph/Samsung_Evo990/Projects/00_AI/00_Workflows/obsidian-skills/` |
| **URL** | [github.com/kepano/obsidian-skills](https://github.com/kepano/obsidian-skills) |
| **What it does** | Collection of Agent Skills (agentskills.io spec) teaching AI agents how to interact with Obsidian vaults. |
| **Skills** (5 total) | `obsidian-markdown` (wikilinks, callouts, embeds, frontmatter), `obsidian-bases` (.base files, YAML views), `json-canvas` (.canvas spec), `obsidian-cli` (CLI wrapper), `defuddle` (web content extraction). |

**Key patterns:**

| Pattern | Description | WP Relevance |
|---------|-------------|--------------|
| **Validation-at-end workflow** | Every skill forces a verify/validate step after mutation. Applies to markdown, bases, canvas. | WP5 (skills — validate output after note creation) |
| **Three interaction modes documented** | Filesystem (direct I/O, no Obsidian needed), CLI (running Obsidian process), external tools (defuddle, no vault required). No MCP used — pure Agent Skills markdown spec. | WP3 (document interaction mode clearly per tool) |
| **Offloaded reference files** (`references/`) | Dense reference material (callout types, function reference, examples) in separate files. SKILL.md stays lean and action-oriented. | WP5 (skill structure — keep SKILL.md lean, reference material in sub-files) |
| **Concrete examples inline** | Every concept has embedded YAML/JSON/bash/markdown examples. Troubleshooting sections for common errors. | WP5 (skill writing — include concrete examples) |

---

#### Auto-claude-code-research-in-sleep (ARIS)

| | |
|---|---|
| **Path** | `/media/christoph/Samsung_Evo990/Projects/00_AI/00_Workflows/Auto-claude-code-research-in-sleep/` |
| **URL** | [github.com/wanshuiyin/Auto-claude-code-research-in-sleep](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep) |
| **What it does** | Composable Markdown-skill framework orchestrating full ML research lifecycle: idea discovery → adversarial cross-model review → compiled PDF. Runs as autonomous overnight agent pipeline. |

**Key patterns:**

| Pattern | Description | WP Relevance |
|---------|-------------|--------------|
| **Artifact contracts between stages** | Skills communicate exclusively through named plain-text files (IDEA_REPORT.md, EXPERIMENT_LOG.md, NARRATIVE_REPORT.md) — never in-chat summaries. Reviewer-independence: pass file paths only. | WP6 (CLI JSON contracts between pipeline stages) |
| **Adversarial review loop** | Up to 4 rounds, external LLM from DIFFERENT model family scores work and demands fixes. Converges at score >= 6/10 or rounds exhausted. | WP6 (evaluator — cross-model review for quality) |
| **Research-wiki skill** | Maintains persistent knowledge base with 4 entity types (papers, ideas, experiments, claims) as individual .md files with structured YAML frontmatter. Typed relationship graph in `graph/edges.jsonl` with 8 edge types. `query_pack.md` is hard-budgeted 8000-char compressed summary. | WP5 (vault-search — structured entity pages with typed edges)
| **Integration contract** | Formalizes 6 required components per cross-skill integration: activation predicate, canonical helper, concrete artifact, visible checklist, repair command, verifier. | WP6 (pipeline — formal integration contracts between stages) |
| **Effort levels** | `lite`/`balanced`/`max`/`beast` scaling token usage parametrically. | WP5 (vault-search — quick/standard/deep modes mapping to QMD search modes) |

---

### Prior Art: personal-os-skills

| | |
|---|---|
| **Path** | `/media/christoph/Samsung_Evo990/Projects/00_AI/00_Workflows/personal-os-skills/` |
| **Relevance** | Source for `recall` and `sync-claude-sessions` skills to be adapted in WP5. 7 skills total from Artem Zhutov. |

---

### Plugin Target

| Item | Value |
|------|-------|
| Plugin repo | `Truncuso/qmd-obsidian` |
| Local path | `/media/christoph/Samsung_Evo990/Projects/00_AI/00_Workflows/qmd-obsidian/` |
| GitHub | [github.com/Truncuso/qmd-obsidian](https://github.com/Truncuso/qmd-obsidian) |

---

## Key Design Insights from Reference Exploration

These patterns discovered during exploration should inform implementation:

1. **Hot cache pattern** (from claude-obsidian) — WP5 vault-search should maintain a `hot.md` of recent vault context, restored on SessionStart, refreshed on PostCompact. This bridges the session-memory gap without scanning the full vault.

2. **Artifact contracts** (from ARIS) — WP6 ingestion stages should pass data through named JSON files, not in-chat. Enables independent testing of each stage, cross-model review, and deterministic replay.

3. **AI-writing quality gate** (from avoid-ai-writing) — WP6 evaluator should include AI-pattern detection as a quality dimension. Ingestion output that smells like AI gets flagged for human review before vault entry.

4. **Validation-at-end** (from obsidian-skills) — Every WP5 skill must end with a verify/validate step. Created notes checked for valid frontmatter, wikilinks checked for dead targets, search results checked for relevance.

5. **Deterministic idempotency** (from claude-obsidian) — WP6 source adapter should generate deterministic IDs from content hashes, not timestamps. Same input → same output path → skip re-ingestion.

6. **Graceful fallback chains** (from claude-obsidian feature detection) — WP3 access layer: try QMD search first, fall back to mcpvault search_notes if QMD down, fall back to Obsidian CLI search if Obsidian is running, error gracefully if all unavailable.

7. **Integration contracts** (from ARIS) — WP6 pipeline stages need formal contracts: what each stage consumes, what it produces, its activation predicate, its verifier, and its repair command.

---

## Acceptance Criteria

- [ ] `grep -r '/home/' ~/.claude/scripts/` returns 0 hardcoded paths (except config.env comments)
- [ ] QMD collections populated: `qmd status --json` shows document_count > 0 for work-vault
- [ ] QMD daemon running: `curl localhost:8181/health` → 200
- [ ] mcpvault functional: `read_note` returns content, `write_note` creates file visible in Obsidian
- [ ] Vault switching: `vault switch test` updates $OBSIDIAN_ACTIVE_VAULT and QMD scope
- [ ] All 5 new skills load and execute correctly
- [ ] Ingestion pipeline produces evaluated, structured markdown notes from YouTube/PDF/podcast sources
- [ ] All verification matrix invariants (19 total) pass
- [ ] Test suites: unit, integration, E2E all green

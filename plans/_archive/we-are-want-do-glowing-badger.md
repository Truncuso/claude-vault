# WP: QMD + Obsidian Vault Integration System

**Status:** specified
**Created:** 2026-05-13
**Author:** Christoph Unger

---

## Context

The user has a mature Claude Code configuration (32 skills, 42 agents, SDD workflow) but two gaps: (1) QMD is installed but has 0 indexed documents — no collections, no search; (2) no Obsidian vault MCP access exists. The goal is a portable, env-var-driven integration of QMD semantic search + Obsidian vault management, with a LangChain/LlamaIndex-based evolution of the draft agentic note ingestion pipeline. The system must manage multiple vaults, support automatic embedding updates, and be verifiable end-to-end.

---

## Architecture Decision: Three-Layer Access Model

After evaluating all options, the recommendation is a three-layer model:

| Layer | Tool | Role | Requires Obsidian Running? |
|-------|------|------|---------------------------|
| **Search** | QMD (MCP) | Semantic + keyword search across vaults. Saves tokens by finding relevant context BEFORE loading files. | No |
| **CRUD** | mcpvault (`npx @bitbonsai/mcpvault`) | MCP-native read/write/patch/delete/move notes, frontmatter, tags, directory listing. 14 tools, token-optimized. | No |
| **Live Ops** | Obsidian CLI (`obsidian search`, `obsidian create`, `obsidian daily:append`) | Template-based note creation, UI open commands, daily note operations. Only when Obsidian IS running. | Yes |

**Why not obsidian-local-rest-api bridge?** mcpvault provides the same CRUD capabilities as MCP tools natively — no bridge server to build/maintain. The REST API plugin remains available as a fallback but mcpvault is the primary CRUD path.

**Why not Obsidian CLI for everything?** The CLI requires Obsidian running (Electron app). For headless/automated workflows (batch ingestion, session sync), QMD + mcpvault work independently.

---

## Phased Implementation Plan

### Phase 0: Foundation — Environment & Path Conventions

**Goal:** Portable config with zero hardcoded paths. All scripts resolve paths from env vars.

**Files to create:**

| File | Purpose |
|------|---------|
| `~/.claude/config.env` | User-editable: defines `VAULTS_BASE`, vault paths, ports, API keys |
| `~/.claude/scripts/lib/env.sh` | Sources `config.env`, validates required vars, provides `resolve_path()` |
| `~/.claude/scripts/lib/vault-registry.sh` | CLI: `vault list`, `vault switch <name>`, `vault create <name>`, `vault info` |
| `~/.claude/scripts/lib/vault-registry.json` | Machine-readable registry of known vaults |

**Files to modify:**

| File | Change |
|------|--------|
| `~/.claude/settings.json` | Fix `/home/cunger/` → use `~` or env var refs |
| `~/.claude/hooks/hooks.json` | Fix hardcoded paths in hook commands |
| `dotfiles/claude/` (git-tracked) | Ensure symlink consistency |

**Env var schema (config.env):**

```bash
# --- Paths (edit per workstation) ---
export VAULTS_BASE="/media/${USER}/M2.SSD1/000_Vaults"
export PROJECTS_BASE="/media/${USER}/Samsung_Evo990/Projects"

# --- Vault registry ---
export OBSIDIAN_WORK_VAULT="${VAULTS_BASE}/Obsidian_Work_Vault"
export OBSIDIAN_TEST_VAULT="${VAULTS_BASE}/Obsidian_Test_Vault"
export OBSIDIAN_PERSONAL_VAULT="${VAULTS_BASE}/Obsidian_Personal_Vault"
export OBSIDIAN_ACTIVE_VAULT="${OBSIDIAN_WORK_VAULT}"

# --- QMD ---
export QMD_INDEX_DIR="${HOME}/.cache/qmd"
export QMD_DAEMON_PORT=8181

# --- Ollama ---
export OLLAMA_BASE_URL="http://127.0.0.1:11434"

# --- Obsidian CLI ---
export OBSIDIAN_VAULT_NAME="Obsidian_Work_Vault"  # name as seen in Obsidian app
```

**Verification:**
- `grep -r '/home/' ~/.claude/scripts/` returns 0 hits (except config.env comments)
- `source ~/.claude/scripts/lib/env.sh` works in clean shell
- `vault list` outputs valid JSON with all registered vaults

**Complexity delta:** +2 shell libs, +1 JSON registry, +1 env file; -5+ scattered hardcoded paths. Net justified by portability.

---

### Phase 1: QMD Configuration — Collections, Indexing, Daemon, Auto-Embed

**Goal:** QMD collections per vault + session transcripts, persistent daemon, automatic embedding updates.

**Collections:**

```
work-vault       → $OBSIDIAN_WORK_VAULT/**/*.md
sessions         → ~/.claude/sessions/*.jsonl
```

Future vaults get collections on creation (Phase 3).

**Files to create:**

| File | Purpose |
|------|---------|
| `~/.claude/scripts/qmd/setup-collections.sh` | Idempotent: create collections, set context descriptions, run `qmd update` + `qmd embed` |
| `~/.claude/scripts/qmd/qmd-daemon.sh` | `start`, `stop`, `status`, `ensure` subcommands for daemon lifecycle |
| `~/.claude/scripts/qmd/auto-embed.sh` | Watches vault dirs via inotify, triggers `qmd embed` on .md changes (debounced 30s) |
| `~/.config/systemd/user/qmd-daemon.service` | systemd user service for QMD HTTP daemon |
| `~/.config/systemd/user/qmd-auto-embed.service` | systemd user service for file watcher |
| `~/.claude/scripts/qmd/contexts/work-vault.md` | Context descriptions for QMD sub-path contexts |

**Files to modify:**

| File | Change |
|------|--------|
| `~/.claude/hooks/hooks.json` | SessionStart hook: run `qmd-daemon.sh ensure` |
| `dotfiles/claude/mcp-configs/mcp-servers.json` | Add QMD HTTP MCP config pointing to `localhost:${QMD_DAEMON_PORT}` |

**Auto-embed strategy:**

```
inotifywait -m -r --format '%w%f' -e modify,create,delete,move \
  "${OBSIDIAN_WORK_VAULT}" --include '\.md$' \
  | while read f; do
      debounce 30s → qmd embed --collection work-vault
    done
```

Also triggers on git pull (if vault is git-tracked) via a post-merge hook template.

**Daemon lifecycle:**
- systemd user service for persistence across reboots
- SessionStart hook as fallback: `qmd-daemon.sh ensure` (starts if not running, no-op if already up)

**Verification:**
- `qmd status --json` shows work-vault collection with document_count > 0
- `qmd search "test" --collection work-vault` returns ranked results
- `curl http://localhost:8181/health` → 200
- MCP tools `mcp__plugin_qmd_qmd__search` / `get` / `status` functional
- Modify a .md file in vault → within 30s, `qmd embed` runs automatically

**Complexity delta:** +4 scripts, +2 systemd units, +1 hook, +1 MCP config. Justified by QMD being the core search infrastructure everything else depends on.

---

### Phase 2: Obsidian MCP Access — mcpvault + CLI

**Goal:** MCP-native vault CRUD via mcpvault, complementary Obsidian CLI for live operations.

**Files to create:**

| File | Purpose |
|------|---------|
| `dotfiles/claude/mcp-configs/mcp-servers.json` | Add mcpvault entry per vault |
| `~/.claude/scripts/obsidian/obsidian-cli.sh` | Wrapper: `obsidian search`, `obsidian create`, `obsidian daily:append` with JSON output parsing |

**mcpvault MCP config (per vault):**

```json
{
  "mcpvault-work": {
    "command": "npx",
    "args": ["-y", "@bitbonsai/mcpvault@latest", "${OBSIDIAN_WORK_VAULT}"],
    "description": "CRUD access to work vault: read/write/patch/delete notes, frontmatter, tags, search"
  }
}
```

Additional vaults get their own mcpvault instance (each pointed at a different vault directory). mcpvault can only serve one vault per instance, so multi-vault requires multiple MCP server entries.

**Obsidian CLI wrapper contract:**

```bash
obsidian-cli search "meeting notes" --vault work --json
# → {"results": [{"path": "...", "score": 0.9, "excerpt": "..."}]}

obsidian-cli create "Trip Report" --template Travel --vault work --json
# → {"created": true, "path": "003_Projects/Trip Report.md"}

obsidian-cli daily:append "- [ ] Review PR" --vault work --json
# → {"appended": true, "path": "System/Daily/2026-05-13.md"}
```

**Verification:**
- `npx @bitbonsai/mcpvault@latest ${OBSIDIAN_WORK_VAULT}` starts and lists 14 tools
- `mcp__mcpvault-work__read_note` returns content for a known vault file
- `mcp__mcpvault-work__write_note` creates a file visible in Obsidian
- `obsidian search "test"` returns results (when Obsidian is running)

**Complexity delta:** +1 MCP config per vault, +1 shell wrapper, -1 custom bridge server (avoided by using mcpvault). Net zero or negative — mcpvault eliminates the need for the originally-planned Python bridge.

---

### Phase 3: Multi-Vault Management

**Goal:** Create the test vault, formalize vault registry, enable vault switching.

**Files to create:**

| File | Purpose |
|------|---------|
| `~/.obsidian-vault-template/` | Skeleton for new vaults: minimal `.obsidian/` config, standard folder structure |
| `~/.claude/scripts/vault/create-vault.sh` | Scaffold new vault from template, init git, register in vault-registry.json, add QMD collection |
| `~/.claude/scripts/vault/test-vault.sh` | Test vault creation + validation |

**New vault: Obsidian_Test_Vault**

Created at `${VAULTS_BASE}/Obsidian_Test_Vault/` with:
- `.obsidian/` — minimal config (templater-obsidian plugin only)
- Standard folders: `000_Projects/`, `Areas/`, `System/`, `attachments/`, `00_raw/`
- Git init
- QMD collection `test-vault` auto-created
- Registered in vault-registry.json

**vault.sh CLI contract:**

```bash
vault list                    # {"vaults": [...], "active": "work"}
vault switch personal         # {"previous": "work", "current": "personal"}
vault create experiment       # {"created": true, "path": "...", "qmd_collection": "experiment-vault"}
vault info --name work        # {"name": "work", "path": "...", "qmd_collection": "work-vault", ...}
```

**Verification:**
- `vault create test` creates a fully functional vault
- Vault opens in Obsidian as a separate vault
- QMD collection auto-added, mcpvault config ready
- `vault switch test` updates `$OBSIDIAN_ACTIVE_VAULT` and all dependent env vars

**Complexity delta:** +2 scripts, +1 template directory. Justified by multi-vault isolation (work vs. personal vs. experimental).

---

### Phase 4: Skills & Templates

**Goal:** Skills for vault operations, search, note creation, and unified ingestion entrypoint.

**New skills:**

| Skill | Trigger | Purpose |
|-------|---------|---------|
| `vault-manager` | "manage vaults", "create vault", "switch vault" | Multi-vault ops via vault.sh |
| `vault-search` | "search vault", "find in notes", "what do I have about" | QMD semantic search → synthesized answer |
| `obsidian-note` | "create note", "write to obsidian" | CRUD via mcpvault + Obsidian CLI |
| `daily-note` | "daily note", "journal entry" | Daily note operations via Obsidian CLI |
| `ingest-content` | "ingest this", "process video", "summarize PDF" | Unified routing to LangChain pipelines |

**Updated skills (from personal-os-skills):**

| Skill | Changes |
|-------|---------|
| `recall` | Fix paths → env vars, add vector search mode, add `--collection` scoping |
| `sync-claude-sessions` | Fix paths, trigger QMD re-index on export, validate frontmatter |

**Skill structure convention:**

Each skill follows the established pattern with additions:
```
skills/<name>/
  SKILL.md              # Frontmatter + instructions
  scripts/              # Shell/Python scripts (if needed)
  templates/            # Note templates for Obsidian output
  workflows/            # LLM workflow definitions (prompt chains, routing)
  tests/                # Skill-specific test scenarios
```

**Verification:**
- All 5 new skills load without error
- `/vault-search "machine learning"` returns QMD results with synthesis
- `/obsidian-note "test"` creates file via mcpvault
- `/ingest-content https://youtube.com/...` routes correctly

**Complexity delta:** +5 skills with full subdirectories, +2 skill modifications. Net justified: skills are the primary user interface.

---

### Phase 5: Agentic Ingestion — LangChain/LlamaIndex Migration

**Goal:** Migrate the draft `agentic_note_ingestion` project from smolagents to LangChain/LlamaIndex, add model discovery, prompt registry, evaluator, and podcast pipeline.

**Architecture (target):**

```
agentic_note_ingestion/
  src/ingestion/
    common/
      config.py              # RuntimeConfig (existing, keep)
      model_discovery.py     # NEW: Ollama + API model enumeration, capability ranking
      prompt_registry.py     # NEW: Versioned prompts from YAML/JSON files
      source_adapter.py      # NEW: Unified ContentSource protocol
      evaluator.py           # NEW: Output quality scoring
    youtube/
      orchestrator.py        # REFACTOR: smolagents → LangChain
      services.py            # REFACTOR: keep extraction, replace agent orchestration
    pdf/
      orchestrator.py        # REFACTOR: smolagents → LangChain
      services.py            # REFACTOR: keep extraction/chunking, replace agent orchestration
    podcast/                 # NEW: podcast ingestion pipeline
      orchestrator.py
      services.py
    cli/                     # NEW: unified CLI
      main.py
  prompts/                   # NEW: versioned prompt templates
    youtube_summary.v1.yaml
    pdf_extraction.v1.yaml
    podcast_notes.v1.yaml
  tests/
    test_model_discovery.py
    test_evaluator.py
    test_podcast_pipeline.py
    test_youtube_orchestrator.py  # UPDATE for LangChain
    test_pdf_services.py          # UPDATE for LangChain
  ARCHITECTURE.md            # UPDATE
  SETUP.md                   # UPDATE
```

**Model Discovery Service:**

```python
@dataclass
class DiscoveredModel:
    name: str
    provider: Literal["ollama", "openai", "anthropic", "groq"]
    context_length: int
    available: bool
    capabilities: list[str]  # ["text", "vision", "tools", "structured_output"]

def discover_models() -> list[DiscoveredModel]:
    """Query Ollama /list, check API keys in env, return ranked models."""
```

**Prompt Registry:**

```yaml
# prompts/youtube_summary.v1.yaml
name: youtube_summary
version: "1.0"
model_constraints:
  min_context: 8192
  recommended_provider: ollama
template: |
  You are analyzing a YouTube video transcript.
  Title: {title}
  Channel: {channel}
  
  ## Transcript
  {transcript}
  
  Produce a structured note with:
  1. Key takeaways (3-5 bullets)
  2. Detailed summary (by topic sections)
  3. Actionable insights
  4. Related concepts (for wiki-linking)
  
  Output format: Markdown with YAML frontmatter.
```

**Source Adapter Protocol:**

```python
class ContentSource(Protocol):
    source_type: str  # "youtube", "pdf", "podcast", "url"
    
    def extract(self, ref: str) -> ExtractionPayload: ...
    def chunk(self, payload: ExtractionPayload) -> list[Chunk]: ...
    def get_metadata(self, ref: str) -> dict: ...
```

**CLI contract (unified entrypoint):**

```bash
# Input
python -m ingestion.cli.main --source youtube --ref "https://..." --evaluate

# Output (stdout JSON)
{
  "status": "success",
  "source": {"type": "youtube", "ref": "https://..."},
  "model_used": {"provider": "ollama", "name": "glm-5:cloud"},
  "artifacts": [
    {"type": "note", "path": "003_Projects/Video Summary - Title.md"},
    {"type": "transcript", "path": "attachments/transcript_abc123.txt"}
  ],
  "evaluation": {
    "structure_compliance": 0.95,
    "completeness": 0.87,
    "hallucination_risk": "low",
    "overall": "pass"
  },
  "timing": {"extraction_ms": 3200, "generation_ms": 12400, "total_ms": 15600}
}
```

**Backward compatibility:**
- Existing Templater templates (`YT_Local_Ingestion_v2.md`, `PDF_Agent_Ingestion.md`) continue working
- New entrypoints added alongside, old ones deprecated but not removed

**Verification:**
- `discover_models()` returns available Ollama models (and API models if keys configured)
- YouTube ingestion with LangChain produces structured markdown note
- PDF pipeline preserves section boundaries correctly
- Podcast pipeline extracts audio transcript and generates structured notes
- All JSON outputs validate against contracts
- Existing verification matrix invariants V1-V8 all pass

**Complexity delta:** +4 new modules, +1 new pipeline, +prompt registry, +evaluator, +CLI; -smolagents dependency. Net justified: LangChain is the user's explicit choice; it provides better ecosystem compatibility and evaluator tooling.

---

### Phase 6: Testing & Verification Framework

**Goal:** Comprehensive test suites at three levels with verification matrices.

**Test structure:**

```
~/.claude/tests/
  unit/
    test_env_resolution.py       # All scripts resolve env vars
    test_vault_registry.py       # vault.sh contract compliance
    test_qmd_scripts.py          # QMD setup/daemon scripts
    test_obsidian_cli.py         # CLI wrapper contract
  integration/
    test_qmd_mcp.py              # QMD MCP tool invocations
    test_mcpvault.py             # mcpvault CRUD roundtrip
    test_skill_invocation.py     # Skill loading + execution
    test_agent_spawn.py          # Agent spawn with known tasks
  e2e/
    test_search_to_note.py       # Full: search → find → retrieve → create note
    test_daily_note_append.py    # Full: daily note append via CLI
    test_ingestion_pipeline.py   # Full: YouTube/PDF ingestion workflow
    test_vault_create_switch.py  # Full: create vault → switch → verify
  fixtures/
    sample_vault/                # Minimal vault with known content
    sample_transcript.txt
    sample_paper.pdf
```

**Verification Matrix (master):**

| ID | Invariant | Method | Phase Gate | Severity |
|----|-----------|--------|------------|----------|
| V0.1 | No hardcoded `/home/` or `/media/` in scripts | grep | P0 | BLOCK |
| V0.2 | env.sh is idempotent | source 2x, diff env | P0 | BLOCK |
| V0.3 | All vault paths resolve correctly | `vault list` JSON check | P0 | BLOCK |
| V1.1 | QMD collections exist, doc count > 0 | `qmd status --json` | P1 | BLOCK |
| V1.2 | QMD daemon responds on port 8181 | `curl /health` | P1 | BLOCK |
| V1.3 | Auto-embed triggers on file change | touch .md, wait 35s, check embed log | P1 | WARN |
| V2.1 | mcpvault lists 14 tools for each vault | MCP list_tools | P2 | BLOCK |
| V2.2 | mcpvault read/write roundtrip | write → read → compare | P2 | BLOCK |
| V2.3 | Obsidian CLI search returns results | `obsidian search` (app running) | P2 | WARN |
| V3.1 | vault create produces valid vault dir | directory structure check | P3 | BLOCK |
| V3.2 | vault switch updates all references | env var + QMD scope check | P3 | BLOCK |
| V4.1 | All 5 new skills load without error | Skill invocation | P4 | BLOCK |
| V4.2 | vault-search returns synthesized results | E2E with known query | P4 | WARN |
| V5.1 | Ingestion produces valid JSON contracts | Schema validation | P5 | BLOCK |
| V5.2 | Model discovery returns available models | count > 0, valid fields | P5 | BLOCK |
| V5.3 | Evaluator produces numeric scores | Output format check | P5 | WARN |
| V5.4 | Backward compat: old templates work | Run existing Templater templates | P5 | BLOCK |
| V6.1 | All unit tests pass | `pytest tests/unit/` | P6 | BLOCK |
| V6.2 | All integration tests pass | `pytest tests/integration/` | P6 | WARN |
| V6.3 | All E2E tests pass | `pytest tests/e2e/` | P6 | WARN |

**Test contract convention:**

All test runners output standardized JSON:
```json
{
  "suite": "unit.test_vault_registry",
  "results": [
    {"test": "test_list_json", "status": "pass", "duration_ms": 45},
    {"test": "test_switch_updates_env", "status": "fail", "error": "..."}
  ],
  "summary": {"total": 12, "pass": 11, "fail": 1, "skip": 0, "coverage_pct": 87}
}
```

**Complexity delta:** +10 test files, +fixtures, +verification matrices. Justified: explicit requirement for three-level testing with evaluatable workflows.

---

## Dependency Graph

```
P0 (Foundation)
  |
  +---> P1 (QMD) -----> P3 (Multi-Vault)
  |        |                 |
  +---> P2 (mcpvault+CLI) --+
           |                 |
           +---> P4 (Skills) ---> P5 (Ingestion)
                                    |
                                    v
                              P6 (Testing) -- validates all phases
```

P1 and P2 are parallel. P3 depends on P1. P4 depends on P1+P2. P5 depends on P4 (ingest-content skill). P6 wraps around everything.

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| QMD indexing fails on very large vault | Medium | Medium | Use `--mask` for selective indexing; test with work vault first |
| mcpvault multi-vault limitation (one per instance) | High | Low | One MCP server entry per vault; documented as known constraint |
| LangChain migration breaks existing ingestion | Medium | High | Backward compat layer; keep old entrypoints; phased migration |
| Obsidian CLI requires app running | High | Medium | Document clearly; mcpvault handles headless operations |
| inotify watch limit exceeded on large vault | Low | Medium | Increase `fs.inotify.max_user_watches`; document in SETUP.md |
| Ollama model not available on all workstations | Medium | Medium | Model discovery falls back gracefully; document required models |
| Hardcoded path regression | Medium | Medium | V0.1 invariant enforced via pre-commit grep check |

---

## Open Questions

1. **API keys:** Should the obsidian-local-rest-api key (currently in vault `.obsidian/`) be rotated and stored in a secrets manager instead of config.env?
2. **Minimal plugin set for new vaults:** Baseline: `templater-obsidian` only? Or also `obsidian-git`, `dataview`?
3. **Vault naming convention:** `Obsidian_<Purpose>_Vault` or simpler `Obsidian_<Purpose>`?
4. **Personal vault:** Create `${VAULTS_BASE}/Obsidian_Personal_Vault` now or later?
5. **TypeScript scope:** The plan keeps Python as the primary ingestion language. Should a TypeScript CLI wrapper be added in Phase 5, or defer?

---

## Recommended Agents

- **Phase 0-3:** `code-pro` — shell scripts, JSON configs, systemd units
- **Phase 4:** `code-pro` + `plan-documenter` — skill templates follow established patterns
- **Phase 5:** `python-pro` — LangChain/LlamaIndex migration, type-safe Python
- **Phase 6:** `tdd-guide` + `python-testing` — test-first for all test suites
- **Review gates:** `code-reviewer` after each phase, `security-reviewer` for MCP configs

---

## Verification (How to Test End-to-End)

1. **Bootstrap:** `source ~/.claude/scripts/lib/env.sh && vault list` — shows registered vaults
2. **Index:** `bash ~/.claude/scripts/qmd/setup-collections.sh` — QMD collections populated
3. **Daemon:** `systemctl --user status qmd-daemon` — running, port 8181
4. **Search:** Use MCP `mcp__plugin_qmd_qmd__search` with test query — returns ranked .md files
5. **CRUD:** Use MCP `mcp__mcpvault-work__write_note` to create a test note — visible in Obsidian
6. **Live:** `obsidian search "test"` (with Obsidian open) — returns results
7. **Skills:** `/vault-search "knowledge graph"` — returns synthesized answer from vault
8. **Ingestion:** `python -m ingestion.cli.main --source youtube --ref "<test_url>" --evaluate` — produces evaluated note
9. **Tests:** `python -m pytest ~/.claude/tests/unit/ && python -m pytest ~/.claude/tests/integration/` — all green
10. **Matrix:** All 19 invariants checked and passing in VERIFICATION_MATRIX.md

# Verification Matrix — claude-vault

**Date**: 2026-05-14

---

## Build Verification Gates (All WPs)

| Gate | Command | Expected |
|------|---------|----------|
| Shell syntax | `for f in .claude/scripts/**/*.sh; do bash -n "$f"; done` | No errors |
| Python syntax | `python -m compileall .claude/scripts/` | No syntax errors |
| JSON validity | `for f in .claude/**/*.json; do python -m json.tool "$f" > /dev/null; done` | All JSON valid |
| YAML validity | `for f in .claude/prompts/**/*.yaml; do python -c "import yaml; yaml.safe_load(open('$f'))"; done` | All YAML valid |

---

## Static Guardrails

| Invariant | Check | Applies To |
|-----------|-------|------------|
| No hardcoded absolute paths | `grep -rn '/home/\|/media/' .claude/scripts/` (exclude config.env) | All scripts |
| Vault-relative paths used | Paths resolve from `$VAULT_ROOT` | All scripts |
| MCP configs reference env vars | `${VAR}` pattern in settings.json | All MCP configs |

---

## WP0: Repository Setup

| ID | Test | Expected | Method |
|----|------|----------|--------|
| V0.1 | Repo URL correct | `Truncuso/claude-vault` | `git remote -v` |
| V0.2 | plugin.json has correct name | "claude-vault" | Read plugin.json |
| V0.3 | Stale plans archived | All old WPs in `_archive/` | `ls plans/_archive/` |
| V0.4 | Setup skill spec complete | Flow documented, 7 phases | Read WP0 |

---

## WP1: Per-Vault QMD Setup

| ID | Test | Expected | Method |
|----|------|----------|--------|
| V1.1 | QMD collection exists with documents | document_count > 0 | `qmd status --json` |
| V1.2 | QMD daemon responds | HTTP 200 | `curl localhost:8181/health` |
| V1.3 | FileChanged triggers qmd update | Re-index within 35s of .md change | `touch test.md`, wait 35s, check update log |
| V1.4 | `qmd embed` works manually | Embeddings generated | Run `qmd embed`, verify vector count > 0 |
| T1.1 | BM25 search returns results | Ranked results | `qmd search "test" --collection <vault>` |
| T1.2 | MCP search tool functional | Returns MCP results | `mcp__plugin_qmd_qmd__search` |
| T1.3 | MCP get retrieves document | Full content returned | `mcp__plugin_qmd_qmd__get` |
| T1.4 | Daemon idempotent | Single process after double ensure | Call ensure twice, verify one PID |
| T1.5 | systemd service active + enabled | `active (running)`, `enabled` | `systemctl --user is-active/is-enabled qmd-daemon` |

---

## WP2: MCP Integration

| ID | Test | Expected | Method |
|----|------|----------|--------|
| V2.1 | mcpvault lists 14 tools | All tools present | MCP `list_tools` |
| V2.2 | mcpvault read/write roundtrip | Written = read content | write → read → compare → delete |
| V2.3 | Obsidian CLI search returns results | Results when Obsidian running | `obsidian search "test"` |
| T2.1 | mcpvault read_note on existing file | Returns known content | Read known vault file, compare hash |
| T2.2 | mcpvault write_note visible in Obsidian | File appears | Create note, check Obsidian UI |
| T2.3 | mcpvault get_frontmatter parses YAML | Structured frontmatter | Read note with known frontmatter |
| T2.4 | mcpvault search_notes BM25 | Known content ranks high | Search for unique phrase |
| T2.5 | mcpvault patch_note preserves other content | Only specified changes applied | Patch frontmatter, verify body unchanged |
| T2.6 | obsidian-cli handles Obsidian not running | Error JSON, no crash | Run with Obsidian closed |
| T2.7 | obsidian-cli create uses template | Template vars resolved | Create from template, inspect output |
| T2.8 | Ollama MCP lists models | `ollama_list` returns models | MCP `ollama_list` |
| T2.9 | Ollama MCP generates text | `ollama_generate` produces output | MCP `ollama_generate` with test prompt |

---

## WP3: General Skills

| ID | Test | Expected | Method |
|----|------|----------|--------|
| V3.1 | All 6 skills load without error | SKILL.md parsed, description displayed | Skill invocation |
| V3.2 | vault-search returns synthesized results | QMD search + LLM synthesis across ≥2 documents. Query: "what do I know about X?" with known multi-doc answer in vault fixture. | E2E with fixture vault containing 3+ docs on shared topic |
| T3.1 | vault-search quick mode returns file list | BM25 ranked results with scores | Search for unique phrase from fixture vault |
| T3.2 | vault-search deep mode synthesizes multi-doc answer | Combined results from ≥3 QMD calls across different collections/paths. Query bridges two domains (e.g., "how does concept-A relate to concept-B"). | Cross-domain query requiring synthesis from multiple documents |
| T3.3 | obsidian-note creates file via mcpvault | Note visible, content correct | Create, read back |
| T3.4 | daily-note appends without overwriting | Both entries present | Append twice, verify concatenation |
| T3.5 | ingest-content routes YouTube correctly | YouTube pipeline invoked | Provide YT URL, verify pipeline log |
| T3.6 | ingest-content routes PDF correctly | PDF pipeline invoked | Provide PDF path, verify pipeline log |
| T3.7 | recall works with vault-relative paths | Search succeeds, no hardcoded path errors | Invoke recall |
| T3.8 | sync-sessions triggers QMD re-index | sessions collection updated | Export session, check qmd status |
| T3.9 | All slash commands registered | Commands appear in help | `/help` |

---

## WP4: Working Vault Skills + Templates

| ID | Test | Expected | Method |
|----|------|----------|--------|
| V4.1 | All 4 working-vault skills load | SKILL.md parsed | Skill invocation |
| V4.2 | daily-journal produces structured entry | All sections filled, valid frontmatter | E2E with test prompts |
| T4.1 | project-tracker extracts project status | Correct status from frontmatter | Run on known project file |
| T4.2 | quick-capture creates inbox note | Note appears in Inbox/ | Capture test content |
| T4.3 | meeting-notes generates structured note | Participants, decisions, actions present | Generate from test meeting |
| T4.4 | All 5 templates installed correctly | Files exist in vault template folder | `test -f` each template |
| T4.5 | Templates produce valid markdown | Valid frontmatter, correct structure | Render each template |
| T4.6 | Notes from templates appear in Obsidian | Visible in file explorer | Manual: check Obsidian UI |

---

## WP5: Styles & Anti-Patterns

| ID | Test | Expected | Method |
|----|------|----------|--------|
| V5.1 | All 5 style profiles parse as valid YAML | No parse errors | YAML validation |
| V5.2 | Style detection returns correct profile | Correct for vault type | Test with each vault type |
| T5.1 | Anti-pattern filter catches known AI-isms | Tier 1 words flagged | Test with AI-generated sample |
| T5.2 | Style transform applies target voice | Output matches profile | A/B: styled vs unstyled output |
| T5.3 | P0 flags block note creation | Note not created when credibility killer found | Test with P0-trigger content |
| T5.4 | Style override via --style flag works | Manual flag takes priority | Test --style flag |
| T5.5 | Style override via frontmatter works | Frontmatter style: field takes priority | Test with style: casual |

---

## WP6: Agentic Ingestion

| ID | Test | Expected | Method |
|----|------|----------|--------|
| V6.1 | Pipelines produce valid JSON contracts | Schema validation passes. Schema at `schemas/ingestion-output.schema.json` defines required fields: status, source, model_used, artifacts[], evaluation, timing. | JSON Schema validation with `check-jsonschema` or `ajv` |
| V6.2 | Model discovery returns available models | count > 0, valid fields | `discover_models()` |
| V6.3 | Evaluator produces numeric scores | All 0-1, overall valid enum | Test with fixture outputs |
| V6.4 | Backward compat: old templates work | Existing Templater templates execute | Run old templates |
| T6.1 | YouTube pipeline produces structured note | Key takeaways, summary, insights | Full pipeline with test video |
| T6.2 | PDF pipeline preserves section boundaries | Markdown headings match PDF structure | Test with multi-section PDF |
| T6.3 | Podcast pipeline transcribes and structures | Transcript + structured note | Test with sample audio |
| T6.4 | Model auto-select falls back gracefully | Uses next-best if preferred unavailable | Kill Ollama, run ingestion |
| T6.5 | Prompt registry loads and renders templates | Jinja2 vars resolved | Unit test |
| T6.6 | Source adapter routes by ref type | youtube.com → YouTubeSource, .pdf → PDFSource | Unit test |
| T6.7 | CLI error handling via JSON contract | error_code, message, details | Run with invalid source |
| T6.8 | Timing info in CLI output | extraction_ms, generation_ms, total_ms > 0 | Parse output |
| T6.9 | Style-aware output respects profile | Output matches target style | Compare casual vs academic from same source |
| T6.10 | Anti-pattern filter runs on ingestion output | AI-isms flagged in evaluation | Ingest known AI-generated content |

---

## Live / CLI Verification (End-to-End)

| ID | Command / Action | Expected Observation |
|----|------------------|---------------------|
| L1 | Run setup skill in empty vault | 7-phase flow, .claude/ populated, QMD indexed |
| L2 | `qmd status --json` | Collections listed, document_count > 0 |
| L3 | `curl localhost:8181/health` | HTTP 200 |
| L4 | MCP `mcp__plugin_qmd_qmd__search` "test" | Returns ranked .md files from vault |
| L5 | MCP `mcp__mcpvault-*__write_note` create test note | Note appears in Obsidian |
| L6 | MCP `mcp__ollama__ollama_list` | Returns installed Ollama models |
| L7 | `/vault-search "machine learning"` | Returns synthesized answer from vault |
| L8 | `/ingest-content <youtube-url> --style professional` | Produces style-formatted structured note |
| L9 | `python -m pytest .claude/tests/unit/ -v` | All unit tests pass |
| L10 | `python -m pytest .claude/tests/integration/ -v` | All integration tests pass |

---

## Verification Status

| WP | Shell Syntax | JSON/YAML Validity | Specific Tests | Live/CLI | Overall |
|----|-------------|-------------------|----------------|----------|---------|
| WP0 | — | — | — | — | specified |
| WP1 | — | — | — | — | specified |
| WP2 | — | — | — | — | specified |
| WP3 | — | — | — | — | specified |
| WP4 | — | — | — | — | specified |
| WP5 | — | — | — | — | specified |
| WP6 | — | — | — | — | specified |
| WP7 | — | — | — | — | draft |
| WP8 | — | — | — | — | draft |

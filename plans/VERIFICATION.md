# Verification Matrix — QMD + Obsidian Vault Integration

**Date**: 2026-05-13

---

## Build Verification Gates (All WPs)

Every work package must pass these gates before being considered complete:

| Gate | Command | Expected |
|------|---------|----------|
| Shell syntax | `for f in ~/.claude/scripts/**/*.sh; do bash -n "$f"; done` | No errors on all .sh files |
| Python syntax | `python -m compileall ~/.claude/scripts/` | No syntax errors |
| JSON validity | `for f in ~/.claude/**/*.json; do python -m json.tool "$f" > /dev/null; done` | All JSON valid |

---

## Static Guardrails

Architecture invariants enforced by grep-based checks:

| Invariant | Check | Applies To |
|-----------|-------|------------|
| No hardcoded `/home/` paths | `grep -r '/home/' ~/.claude/scripts/` (exclude config.env) | All scripts |
| No hardcoded `/media/` paths | `grep -r '/media/' ~/.claude/scripts/` (exclude config.env) | All scripts |
| Env vars used for vault paths | `grep -r 'OBSIDIAN_.*VAULT\|VAULTS_BASE' ~/.claude/scripts/` | All scripts |
| MCP configs reference env vars | `${...}` pattern in mcp-servers.json | All MCP configs |

---

## WP1: Foundation — Environment & Path Conventions

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V0.1 | No hardcoded `/home/` or `/media/` in scripts | 0 grep hits (exclude config.env) | `grep -r '/home/\|/media/' ~/.claude/scripts/` |
| V0.2 | env.sh is idempotent | No duplicate env vars | Source twice, diff `env` output |
| V0.3 | Vault paths resolve correctly | All paths in registry exist | `vault list`, `test -d` each path |
| W1-T1 | config.env syntax valid | No errors | `bash -n ~/.claude/config.env` |
| W1-T2 | vault list returns valid JSON | Valid JSON, active field, vaults array | `vault list \| python -m json.tool` |
| W1-T3 | vault switch updates OBSIDIAN_ACTIVE_VAULT | Env var changes, previous value logged | Source env.sh, vault switch, echo $OBSIDIAN_ACTIVE_VAULT |
| W1-T4 | settings.json has no /home/cunger/ | 0 hits | `grep '/home/cunger' ~/.claude/settings.json` |
| W1-T5 | hooks.json has no /home/cunger/ | 0 hits | `grep '/home/cunger' ~/.claude/hooks/hooks.json` |

---

## WP2: QMD Configuration — Collections, Indexing, Daemon, Auto-Update

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V1.1 | QMD collections exist with documents | document_count > 0 per collection | `qmd status --json` |
| V1.2 | QMD daemon responds on port $QMD_DAEMON_PORT | HTTP 200 | `curl -s localhost:${QMD_DAEMON_PORT}/health` |
| V1.3 | Auto-update triggers on .md file change | `qmd update` runs within 35s of .md change | `touch test.md` in vault, wait 35s, check update log |
| V1.4 | `qmd embed` works when run manually | Vector count > 0 after embed | Run `qmd embed`, verify `qmd status --json` shows vectors |
| W2-T1 | BM25 keyword search returns ranked results | Results array with scores | `qmd search "machine learning" --collection work-vault` |
| W2-T2 | MCP search tool returns results | Non-empty results | `mcp__plugin_qmd_qmd__search` via MCP |
| W2-T3 | MCP get tool retrieves full document | Content matches file on disk | `mcp__plugin_qmd_qmd__get` with known path |
| W2-T4 | Context descriptions improve relevance | Scoped search returns domain-relevant results | A/B: scoped vs unscoped search |
| W2-T5 | systemd service is active | `active (running)` | `systemctl --user is-active qmd-daemon` |
| W2-T6 | systemd service enabled for reboot | `enabled` | `systemctl --user is-enabled qmd-daemon` |
| W2-T7 | SessionStart hook idempotent | No duplicate daemon process | Call `qmd-daemon.sh ensure` twice, verify single PID |

---

## WP3: Obsidian MCP Access — mcpvault + CLI

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V2.1 | mcpvault lists 14 tools | All expected tools present | MCP `list_tools` for mcpvault-work |
| V2.2 | mcpvault read/write roundtrip | Written content = read content | write_note → read_note → compare → delete_note |
| V2.3 | Obsidian CLI search returns results | Results when Obsidian running | `obsidian-cli search "test" --json` |
| W3-T1 | mcpvault read_note on existing file | Returns known content | Read known vault file, compare hash |
| W3-T2 | mcpvault write_note creates file visible in Obsidian | File appears in Obsidian file explorer | Manual: check Obsidian UI after write |
| W3-T3 | mcpvault get_frontmatter parses YAML | Structured frontmatter dict | Read note with known frontmatter, check fields |
| W3-T4 | mcpvault search_notes BM25 relevance | Known content ranks high | Search for unique phrase, verify top result |
| W3-T5 | mcpvault patch_note preserves other content | Only specified changes applied | Patch frontmatter, verify body unchanged |
| W3-T6 | obsidian-cli handles Obsidian not running | Error JSON, exit code non-zero but no crash | Run `obsidian-cli search "test"` with Obsidian closed |
| W3-T7 | obsidian-cli create uses template | Template variables resolved in output | Create from template, inspect note content |

---

## WP4: Multi-Vault Management

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V3.1 | vault create produces valid vault | All standard folders, .obsidian/, .git/ present | `test -d` for each expected directory |
| V3.2 | vault switch updates all references | OBSIDIAN_ACTIVE_VAULT + QMD scope change | Switch, verify env var, verify qmd collection |
| W4-T1 | vault create registers in vault-registry.json | New vault in `vault list` output | Create vault, run `vault list`, grep |
| W4-T2 | vault create adds QMD collection | Collection exists, doc count > 0 | `qmd status --json` after create |
| W4-T3 | vault create initializes git repo | .git/ present, initial commit exists | `git -C <vault_path> log --oneline` |
| W4-T4 | vault create adds mcpvault MCP config | Config entry present in file | grep mcp-servers.json for new entry |
| W4-T5 | vault create refuses to overwrite | Error on duplicate name | `vault create test` twice, check error on second |
| W4-T6 | vault delete --force removes cleanly | Directory gone, registry updated, QMD collection dropped | Create temp vault, delete, verify all traces gone |
| W4-T7 | New vault opens in Obsidian | App recognizes vault | Manual: File → Open Vault → navigate to new dir |

---

## WP5: Skills & Templates

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V4.1 | All 5 new skills load without error | SKILL.md parsed, description displayed | Skill invocation for each |
| V4.2 | vault-search returns synthesized results | QMD search + LLM synthesis from vault | E2E: known query → expected content |
| W5-T1 | vault-manager routes to vault.sh | Correct subcommand executed | Invoke skill, verify vault.sh called |
| W5-T2 | vault-search quick mode returns file list | BM25 results ranked by relevance | Search for known unique content |
| W5-T3 | vault-search deep mode synthesizes multi-doc answer | Combined results from multiple QMD calls | Complex cross-topic query |
| W5-T4 | obsidian-note creates file via mcpvault | Note visible, content correct | Create, read back via mcpvault |
| W5-T5 | daily-note appends without overwriting | Both entries present in daily note | Append twice, verify concatenation |
| W5-T6 | ingest-content routes YouTube URL correctly | YouTube orchestrator invoked | Provide YT URL, verify pipeline log |
| W5-T7 | ingest-content routes PDF path correctly | PDF orchestrator invoked | Provide PDF path, verify pipeline log |
| W5-T8 | recall skill works with env vars | Search succeeds, no hardcoded path errors | Invoke recall, verify results from QMD |
| W5-T9 | sync-claude-sessions triggers QMD re-index | sessions collection doc count increases | Export session, check qmd status |
| W5-T10 | All 5 slash commands registered | Commands appear in help/list | `/help` or check commands/ directory |

---

## WP6: Agentic Ingestion — LangChain/LlamaIndex Migration

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V5.1 | Ingestion produces valid JSON contracts | Schema validation passes | JSON Schema against CLI output |
| V5.2 | Model discovery returns available models | count > 0, valid fields, sorted by context_length | `discover_models()` call |
| V5.3 | Evaluator produces numeric scores | All scores 0-1 range, overall in {pass,fail,needs_review} | Test with fixture outputs |
| V5.4 | Backward compat: old Templater templates work | Existing templates execute, produce notes | Run `YT_Local_Ingestion_v2.md` and `PDF_Agent_Ingestion.md` |
| W6-T1 | YouTube LangChain pipeline = smolagents quality | Structure compliance comparable or better | A/B on same test video |
| W6-T2 | PDF pipeline preserves section boundaries | ## headings match PDF structure | Test with known multi-section PDF |
| W6-T3 | Podcast pipeline: audio → transcript → note | Transcript file created, structured note generated | Test with sample audio file |
| W6-T4 | Model auto-select falls back gracefully | Uses next-best model if preferred unavailable | Kill Ollama, run ingestion |
| W6-T5 | Prompt registry loads and renders templates | Variables substituted correctly | Unit test with fixture data |
| W6-T6 | Source adapter routes by ref type | youtube.com → YouTubeSource, .pdf → PDFSource | Unit test with various refs |
| W6-T7 | CLI error handling via JSON contract | error_code, message, details in output | Run with invalid/missing source |
| W6-T8 | Timing info present in CLI output | extraction_ms, generation_ms, total_ms > 0 | Parse output, verify fields |

---

## Live / CLI Verification

| ID | Command / Action | Expected Observation |
|----|------------------|---------------------|
| L1 | `source ~/.claude/scripts/lib/env.sh && vault list` | JSON array with work vault (and test/personal if created) |
| L2 | `bash ~/.claude/scripts/qmd/setup-collections.sh` | Creates collections, runs update + embed, prints document counts |
| L3 | `systemctl --user start qmd-daemon && curl localhost:8181/health` | HTTP 200 |
| L4 | Use MCP `mcp__plugin_qmd_qmd__search` with "knowledge graph" | Returns ranked .md files from vault |
| L5 | Use MCP `mcp__mcpvault-work__write_note` create test note | Note appears in Obsidian |
| L6 | Open Obsidian, run `/vault-search "machine learning"` | Returns synthesized answer from vault content |
| L7 | `python -m ingestion.cli.main --source youtube --ref "<test_url>" --evaluate` | Produces structured note with evaluation scores |
| L8 | `python -m pytest ~/.claude/tests/unit/ -v` | All unit tests pass |
| L9 | `python -m pytest ~/.claude/tests/integration/ -v` | All integration tests pass (with services running) |
| L10 | `python -m pytest ~/.claude/tests/e2e/ -v` | All E2E tests pass (full stack) |

---

## Verification Status

| WP | Shell Syntax | JSON Validity | Specific Tests | Live/CLI | Review | Overall |
|----|-------------|---------------|----------------|----------|--------|---------|
| WP1 | — | — | — | — | — | specified |
| WP2 | — | — | — | — | — | specified |
| WP3 | — | — | — | — | — | specified |
| WP4 | — | — | — | — | — | specified |
| WP5 | — | — | — | — | — | specified |
| WP6 | — | — | — | — | — | specified |

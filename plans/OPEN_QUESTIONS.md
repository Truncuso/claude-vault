# Open Questions — claude-vault

**Date**: 2026-05-14

---

## Active Questions

### Q1: ARIS Skill Scope for Research Vault (WP7)

**Severity:** HIGH | **Blocks:** WP7

Which ARIS skills are truly essential for a research vault? The current estimate is 8 core skills but this needs auditing:
- Which skills are dependencies of the core set?
- Which Codex-native skills need Claude Code equivalents?
- Can unused skills be excluded from installation to reduce clutter?

---
### Q2: Research Vault Folder Structure (WP7)

**Severity:** MEDIUM | **Blocks:** WP7

Should the research vault use ARIS's `research-wiki/` layout or integrate with the working vault's `000_Projects/` + `Areas/` structure? Mixed approach?

---
### Q3: Ollama Model Minimum for Reviewer Quality (WP7)

**Severity:** MEDIUM | **Blocks:** WP7 (design)

Which Ollama models provide sufficient reviewer quality for the adversarial review loop?
- Minimum recommendation: deepseek-v3.2 (~70B params)?
- Acceptable fallbacks: qwen3.5, gemini-3-pro?
- Can quantized models work for review tasks?

---
### Q4: API Key Storage (WP1)

**Severity:** LOW | **Blocks:** — (not blocking, deferred)

Should API keys (Obsidian REST API, OpenAI, Anthropic) live in `.claude/config/config.env` (gitignored) or in a system keyring/secret manager?

---
### Q5: Obsidian Plugin Auto-Install (WP0 setup skill)

**Severity:** LOW | **Blocks:** — (not blocking)

Can the setup skill auto-install Obsidian community plugins (templater-obsidian, dataview) or must the user install them manually through Obsidian's UI? Affects setup skill flow.

---
### Q6: Learning Vault — Spaced Repetition Integration (WP8)

**Severity:** LOW | **Blocks:** WP8 (design deferrable)

Should spaced repetition integrate with Anki/AnkiConnect or use custom note metadata + QMD queries?

---
### Q7: Course Syllabus Parsing (WP8)

**Severity:** LOW | **Blocks:** WP8 (design deferrable)

For learning vaults: auto-parse syllabi from PDF/web or manual entry?

---

## Resolved Questions

| Question | Resolution | Date |
|----------|-----------|------|
| Global plugin vs per-vault? | Per-vault (`.claude/` inside vault) | 2026-05-14 |
| Plugin name? | `claude-vault` | 2026-05-14 |
| Ollama MCP: build or use existing? | Use `rawveg/ollama-mcp` | 2026-05-14 |
| Old Obsidian_Work_Vault: migrate or keep? | Keep as-is, create new vaults | 2026-05-14 |
| Repo: rename or delete? | Rename Truncuso/qmd-obsidian → Truncuso/claude-vault | 2026-05-14 |
| Install method: marketplace or manual? | Both — marketplace for skill loading, setup script for bootstrapping | 2026-05-14 |
| Smolagents vs LangChain? | LangChain/LlamaIndex (user's explicit choice from original session) | 2026-05-13 |
| mcpvault vs custom bridge? | mcpvault (MCP-native, zero bridge code) | 2026-05-13 |
| QMD auto-update: global or per-collection? | Global only (`qmd update`/`qmd embed` have no `--collection` flag) | 2026-05-14 |

---

## Escalation Protocol

1. If a question blocks a WP for >1 session → escalate to decision with rationale documented
2. If insufficient information → make documented assumption, mark as `ASSUMPTION:`, revisit
3. If question resolves during implementation → move to Resolved table

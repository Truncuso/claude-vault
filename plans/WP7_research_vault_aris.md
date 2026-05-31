# WP7: Research Vault — ARIS Full Integration

**Status**: draft (specified for later, open questions remain)
**Severity**: MEDIUM
**Created**: 2026-05-14
**Depends on**: WP5, WP6

**Skill inheritance:** Research vault includes ALL WP3 general skills (vault-search, obsidian-note, daily-note, ingest-content, recall, sync-sessions) plus the research-tier skills below. WP4 working-vault skills (daily-journal, project-tracker, quick-capture, meeting-notes) are NOT inherited — research vaults have different daily workflows.

---

## Problem

Research-oriented vaults need paper management, literature survey, adversarial review, and paper writing capabilities. The ARIS project (65 skills) provides all of this but uses Codex MCP as primary reviewer. We need to adapt it for Ollama MCP, scope it to essential skills, and integrate it into the claude-vault skill tier system.

---

## Scope Decision

**Not all 65 ARIS skills are needed.** Core subset for research vault:

| ARIS Skill | Role | Priority |
|------------|------|----------|
| `research-wiki` | Persistent paper/idea/experiment/claim knowledge base with typed edges | CORE |
| `auto-review-loop` | 3-tier adversarial review (medium/hard/nightmare) | CORE |
| `paper-reading` | Structured paper notes from PDFs | CORE |
| `lit-survey` | Literature search and survey generation | CORE |
| `paper-writing` | Plan → Figures → LaTeX → Compile → Improvement loop | HIGH |
| `idea-discovery` | Lit survey → Ideation → Novelty check → Pilot | HIGH |
| `result-to-claim` | Update claim status, create experiments, add edges | MEDIUM |
| `idea-creator` | Read query_pack, generate ideas, write back | MEDIUM |

---

## Ollama MCP Adaptation

ARIS uses `mcp__codex__codex` for reviewer calls. We adapt this to use `mcp__ollama__ollama_chat`:

| ARIS Original | Ollama MCP Replacement |
|---------------|----------------------|
| `mcp__codex__codex` | `mcp__ollama__ollama_chat` |
| Model: gpt-5.5 xhigh | Model: any Ollama model (deepseek-v3.2, qwen3.5, gemini-3-pro) |
| Reviewer independence by model family | Reviewer independence by model family (use different Ollama model than executor) |

**Reviewer routing for research vault:**
```
Default: Ollama MCP with deepseek-v3.2 (or user-configured model)
Optional: Codex MCP (if user has access)
External: gemini-review MCP server from ARIS (if API key configured)
```

---

## Open Questions (Blockers)

1. **Which ARIS skills are truly essential?** The 8 listed above are an estimate. Need to audit which skills are dependencies of the core set.
2. **How to handle ARIS's Codex-native skills?** ARIS has `skills-codex/` with ~50 Codex-native variants. These need Claude Code equivalents.
3. **Research vault folder structure:** Should it use ARIS's `research-wiki/` layout or integrate with the working vault's `000_Projects/` + `Areas/` structure?
4. **ARIS MCP servers:** Which of the 6 ARIS MCP servers (claude-review, gemini-review, llm-chat, minimax-chat, codex-image2, feishu-bridge) should be included? Recommend: gemini-review (optional) + llm-chat (for API model fallback).
5. **Installation complexity:** ARIS's `install_aris.sh` symlinks tools into `.aris/`. How does this interact with claude-vault's `.claude/` bootstrap?
6. **Ollama model requirements:** Which Ollama models are sufficient for reviewer quality? Minimum recommendation: deepseek-v3.2 (or equivalent ~70B param model).

---

## Architecture Notes

- ARIS's integration contract pattern (6 required components) should be adopted for all WP7+WP8 inter-skill integrations
- ARIS's effort levels (lite/balanced/max/beast) map naturally to vault-search modes (quick/standard/deep)
- ARIS's `query_pack.md` (8000-char compressed summary) is an excellent pattern for vault-search's hot cache
- ARIS's trace system (`.aris/traces/`) aligns with the traceability requirement — adopt for research vault operations

---

## Verification (When Implemented)

- [ ] `research-wiki` skill initializes wiki structure
- [ ] `paper-reading` generates structured paper note from PDF
- [ ] `auto-review-loop` runs with Ollama MCP as reviewer
- [ ] `lit-survey` produces survey from search results
- [ ] All integration contracts satisfied (6 components each)
- [ ] Traces written for all reviewer calls

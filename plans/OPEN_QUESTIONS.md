# Open Questions — QMD + Obsidian Vault Integration

**Date**: 2026-05-13
**Updated**: 2026-05-13

---

## Active Questions

| ID | Question | Context | Blocks | Owner | Status |
|----|----------|---------|--------|-------|--------|
| Q1 | Should the obsidian-local-rest-api API key be rotated and stored in a secrets manager instead of config.env? | Current key is in vault `.obsidian/plugins/obsidian-local-rest-api/data.json`. If we use REST API as fallback for mcpvault, key management matters. | WP3 (minor — only if REST API used as fallback) | Christoph | open |
| Q2 | What is the minimal plugin set for new vault templates? | Baseline proposal: `templater-obsidian` only. Alternatives: also `obsidian-git`, `dataview`, `obsidian-local-rest-api`? | WP4 | Christoph | open |
| Q3 | Vault naming convention: `Obsidian_<Purpose>_Vault` or `Obsidian_<Purpose>`? | Current: `Obsidian_Work_Vault`. Proposal for new vaults: `Obsidian_Test_Vault`, `Obsidian_Personal_Vault`. Simpler alternative: `Obsidian_Work`, `Obsidian_Test`, `Obsidian_Personal`. | WP4 | Christoph | open |
| Q4 | Create personal vault (`Obsidian_Personal_Vault`) now or defer? | User mentioned "multiple vaults for multiple purposes" but only explicitly requested the test vault. Personal vault can be created alongside test vault or deferred. | WP4 | Christoph | open |
| Q5 | Should ingestion CLI include a TypeScript wrapper, or keep Python-only? | Current ingestion is Python-only. User mentioned "python/typescript or combination". A TS wrapper could provide better Claude Code integration (Node.js native) but adds complexity. | WP6 | Christoph | open |

---

## Resolution Log

| ID | Question | Resolution | Resolved By | Date |
|----|----------|------------|-------------|------|
| Q-ARCH-1 | obsidian-local-rest-api bridge vs mcpvault vs filesystem MCP? | Use mcpvault as primary CRUD (MCP-native, no plugin), Obsidian CLI for live ops, keep REST API as fallback | Christoph | 2026-05-13 |
| Q-ARCH-2 | smolagents vs LangChain/LlamaIndex for ingestion? | LangChain/LlamaIndex per user's explicit choice | Christoph | 2026-05-13 |
| Q-ARCH-3 | QMD daemon: systemd vs SessionStart hook? | Both — systemd for persistence, SessionStart hook as fallback | Christoph | 2026-05-13 |

---

## Escalation Protocol

If any question remains unresolved after 2 rounds of clarification:

1. Halt implementation.
2. Escalate to Christoph with:
   - The unresolved question(s)
   - Impact on dependent WPs
   - Proposed resolution or decision checkpoint (Option 1 / Option 2)
3. Do not proceed past Spec until resolved.

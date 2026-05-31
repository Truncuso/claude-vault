# claude-vault — Vault-Centric Claude Code Plugin

Claude Code plugin that transforms Obsidian vaults into self-contained AI-assisted knowledge environments. Each vault gets its own `.claude/` directory with skills, QMD search, mcpvault CRUD, and Ollama MCP for local models — tailored to the vault's purpose.

**Status**: architecture complete — implementation pending

## Vault Types

| Type | Purpose | Status |
|------|---------|--------|
| **Working Vault** | Daily notes, projects, meetings, general knowledge | WP3-WP5 (pending) |
| **Research Vault** | Paper wiki, literature survey, adversarial review, paper writing | WP7 (draft) |
| **Learning Vault** | Courses, books, concept mapping, spaced repetition | WP8 (draft) |

## Architecture

Four-layer access model within each vault:

| Layer | Tool | Requires Obsidian? |
|-------|------|-------------------|
| Search | QMD per vault (BM25 + vector) | No |
| CRUD | mcpvault per vault (14 tools) | No |
| Live Ops | Obsidian CLI | Yes |
| Local LLM | rawveg/ollama-mcp (14 tools) | No (Ollama must run) |

## Installation

**Marketplace:** `claude plugins install claude-vault` — loads skills. Interactive setup skill runs on first session.

**Manual:** Clone repo into vault, run `./setup.sh` — self-contained bootstrap.

## Structure (within a vault)

```
.claude/
├── CLAUDE.md
├── settings.json         # MCP configs (QMD, mcpvault, Ollama)
├── hooks/hooks.json
├── monitors/monitors.json
├── skills/
│   ├── general/          # All vault types (6 skills)
│   └── <vault-type>/     # Type-specific skills
├── scripts/
├── prompts/styles/       # Note style profiles
└── config/
```

## References

- [QMD](https://github.com/tobi/qmd) — local markdown search engine (by Tobias Lutke)
- [mcpvault](https://github.com/bitbonsai/mcpvault) — MCP-native vault CRUD
- [ollama-mcp](https://github.com/rawveg/ollama-mcp) — Ollama MCP server (14 tools)
- [ARIS](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep) — ML research automation (65 skills)
- [Obsidian CLI](https://obsidian.md/cli) — CLI reference
- [Claude Code Plugins](https://code.claude.com/docs/en/plugins)
- [Claude Code Hooks](https://code.claude.com/docs/en/hooks-guide)

## License

MIT

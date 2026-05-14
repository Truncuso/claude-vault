# qmd-obsidian — QMD + Obsidian Vault Integration Plugin

Claude Code plugin integrating QMD semantic search with Obsidian vault management. Portable, env-var-driven, multi-vault.

**Status**: spec complete — implementation pending

## Architecture

Three-layer access model:

| Layer | Tool | Role | Requires Obsidian Running? |
|-------|------|------|---------------------------|
| Search | QMD (MCP) | BM25 + vector search, saves tokens by finding context before loading files | No |
| CRUD | mcpvault | Read/write/patch/delete notes, frontmatter, tags — 14 tools | No |
| Live Ops | Obsidian CLI | Template creation, daily notes, UI commands | Yes |

## Key Decisions

See `plans/OVERVIEW.md` for full decision log. Summary:

- **mcpvault over custom bridge**: MCP-native, zero bridge code, works headless
- **QMD auto-update split**: `qmd update` via inotify (fast file scan), `qmd embed` via manual/cron (expensive ML inference). Both commands are global — no `--collection` flag exists
- **LangChain/LlamaIndex for ingestion**: User's explicit choice over smolagents
- **Env-var-based paths**: Zero hardcoded absolute paths, `config/config.env` is single source of workstation-specific config
- **Multi-vault from start**: Vault registry + creation scripts in WP4

## Plugin Structure

```
qmd-obsidian/
├── .claude-plugin/plugin.json    # Plugin manifest
├── hooks/hooks.json              # 5 lifecycle hooks
├── monitors/monitors.json        # 2 background monitors
├── settings.json                 # Plugin settings
├── config/
│   ├── config.env.example        # Workstation config template (git tracked)
│   └── vault-registry.json       # Known vaults registry
├── plans/                        # Full SDD documentation (12 files)
├── docs/                         # Architecture + setup guides (pending)
├── bin/                          # CLI executables (pending)
├── scripts/                      # Shell/Python scripts (pending)
└── skills/                       # 5 skills (pending)
```

## Work Packages

| WP | Title | Status |
|----|-------|--------|
| WP1 | Foundation — Environment & Path Conventions | specified |
| WP2 | QMD Configuration — Collections, Indexing, Daemon, Auto-Update | specified |
| WP3 | Obsidian MCP Access — mcpvault + CLI | specified |
| WP4 | Multi-Vault Management | specified |
| WP5 | Skills & Templates | specified |
| WP6 | Agentic Ingestion — LangChain/LlamaIndex Migration | specified |

## Setup (Post-Implementation)

```bash
# 1. Install the plugin
claude plugins install Truncuso/qmd-obsidian

# 2. Copy and edit workstation config
cp config/config.env.example config/config.env
$EDITOR config/config.env

# 3. Run QMD setup (manual, one-time)
bash scripts/qmd/setup-collections.sh

# 4. Start daemon
systemctl --user enable --now qmd-daemon
```

## References

- [QMD](https://github.com/tobi/qmd) — local markdown search engine
- [mcpvault](https://github.com/bitbonsai/mcpvault) — MCP-native vault CRUD
- [Obsidian CLI](https://obsidian.md/cli) — search, create, daily notes
- [Claude Code Plugins](https://code.claude.com/docs/en/plugins)
- [Claude Code Hooks](https://code.claude.com/docs/en/hooks-guide)
- [Blog: personal OS skills](https://x.com/ArtemXTech/status/2028330693659332615)

## License

MIT

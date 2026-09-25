# claude-skills

Personal [Claude Code](https://claude.ai/code) skills, distributed as a plugin marketplace.

## Install

```bash
claude plugin marketplace add vkotaru/claude-skills
claude plugin install web-app-patterns@vkotaru-skills
claude plugin install self-hosted-mcp@vkotaru-skills
```

Restart Claude Code, or start a new session, and the skills load automatically.

To update later:

```bash
claude plugin marketplace update vkotaru-skills
claude plugin update web-app-patterns
claude plugin update self-hosted-mcp
```

## Plugins

### `web-app-patterns`

Opinionated, end-to-end architecture recipes for small web apps.

| Skill | What it does |
|---|---|
| `offline-first-pwa` | Build or retrofit an installable PWA that works fully offline: Dexie local DB, a `pendingSync` mutation queue, a bidirectional sync engine, service-worker update prompt, and the mobile baseline. Includes both server dialects (Express/SQLite and FastAPI/SQLAlchemy) and a verification checklist. |

### `self-hosted-mcp`

Exposing a privately hosted app to Claude without handing the internet a route
into it.

| Skill | What it does |
|---|---|
| `mcp-connector` | Turn a self-hosted, tailnet-only app into a custom remote MCP connector. Covers the separate internet-facing service and why it must not share the app's dependency tree, OAuth 2.1 with DCR and audience-bound rotating tokens, a read-only tool layer built on one query chokepoint, Tailscale Funnel and the ACL that stops the public node reaching anything else, a least-privilege database role, a verification checklist, and eleven traps that each cost real debugging time. |

## Local development

Point Claude Code at a working copy instead of the published repo:

```bash
claude plugin marketplace add /path/to/claude-skills
```

Validate before committing:

```bash
claude plugin validate .
claude plugin validate ./plugins/web-app-patterns
claude plugin validate ./plugins/self-hosted-mcp
```

## Layout

```
.claude-plugin/marketplace.json     the marketplace others add
plugins/<plugin>/
  .claude-plugin/plugin.json        the plugin manifest
  skills/<skill>/SKILL.md           one directory per skill
```

Adding a skill means adding a directory under an existing plugin's `skills/`.
Adding a plugin means a new entry in `marketplace.json` alongside it.

## License

MIT

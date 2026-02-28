# N8N Workflow Creator

A composable skill library that teaches coding agents how to **build, test, and debug n8n workflows entirely via the REST API** — no browser UI required.

Built from real-world experience automating lead generation, email outreach, and data pipelines on n8n Cloud.

## What It Does

Give your coding agent superpowers for n8n:

- **Build workflows programmatically** — create nodes, wire connections, deploy via the n8n REST API
- **Test end-to-end** — trigger forms/webhooks, poll executions, validate node-by-node output
- **Debug failures** — trace data flow, find field mismatches, fix silent data loss
- **Integrate Google Sheets** — append, update, clear, batch operations with proper rate limiting
- **Generate AI content** — use OpenRouter/LLMs in Code nodes with parallelization for the 60s timeout
- **Build email outreach systems** — Hunter.io verification, contact ranking, personalized cold emails

## Skills

| Skill | Description |
|-------|-------------|
| `n8n-workflow-building` | Create and wire n8n workflows via REST API. Node types, connection format, deployment pattern. |
| `n8n-workflow-testing` | End-to-end testing: activate → trigger → poll → validate. Form triggers, webhook triggers, execution monitoring. |
| `n8n-workflow-debugging` | Systematic debugging: fetch workflow, map connections, trace data flow, find field mismatches. |
| `n8n-api-patterns` | n8n REST API patterns: read-modify-write, activation cycling, Code node capabilities and limits. |
| `n8n-google-sheets` | Google Sheets integration: append, update, create tabs, batch operations, cleanup patterns. |
| `n8n-email-outreach` | Hunter.io verification, contact ranking, AI email generation, mail merge setup. |

## Installation

### Claude Code

```
/plugin marketplace add <owner>/n8n-workflow-creator-marketplace
/plugin install n8n-workflow-creator@n8n-workflow-creator-marketplace
```

### Cursor

```
/plugin-add n8n-workflow-creator
```

### Codex

```
Fetch and follow instructions from https://raw.githubusercontent.com/<owner>/N8N-Workflow-Creator/refs/heads/main/.codex/INSTALL.md
```

### OpenCode

```
Fetch and follow instructions from https://raw.githubusercontent.com/<owner>/N8N-Workflow-Creator/refs/heads/main/.opencode/INSTALL.md
```

### GitHub Copilot (Custom Agent)

Copy the skills into your repo's `.github/agents/` directory or reference them in your agent configuration.

### Manual

```bash
git clone https://github.com/<owner>/N8N-Workflow-Creator.git
# Then symlink or copy skills/ into your agent's skill directory
```

## Prerequisites

Your agent environment needs:
- **Python 3.8+** with `requests` (for n8n API calls)
- **n8n instance** (Cloud or self-hosted) with API key
- **Playwright MCP** (optional, for browser-based validation)

## Philosophy

- **API-first** — never rely on the browser UI; everything is reproducible via REST
- **Defensive coding** — accept both old and new field names, validate before writing
- **Test every change** — trigger → poll → validate, never assume 200 = success
- **Batch safely** — respect rate limits, use parallel Promise.all for LLM calls, sequential batches for Sheets

## Contributing

1. Fork the repository
2. Create a branch for your skill
3. Follow the superpowers `writing-skills` format (YAML frontmatter + structured markdown)
4. Submit a PR

## License

MIT License — see [LICENSE](LICENSE) file for details.

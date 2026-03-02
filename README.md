# N8N Workflow Creator

Installable skills that teach coding agents to **build, test, and debug n8n workflows via the REST API** — no browser UI needed. Built from battle-tested production workflows.

## What You Get

| Skill | What It Teaches |
|-------|-----------------|
| **n8n-full-pipeline-setup** | End-to-end setup: environment validation → credentials → build → deploy → test → verify |
| **n8n-workflow-building** | Create nodes, wire connections, deploy workflows programmatically |
| **n8n-workflow-testing** | Trigger → poll → validate end-to-end, because 200 ≠ success |
| **n8n-workflow-debugging** | Trace data flow, find field mismatches, fix silent data loss |
| **n8n-api-patterns** | Code node HTTP, LLM parallelization, 60s timeout workarounds |
| **n8n-google-sheets** | Append, update, batch ops, cleanup, rate limiting |
| **n8n-email-outreach** | Hunter.io verification, contact ranking, AI cold emails, mail merge |

## Install

> **Full install guide with all options:** [docs/INSTALLATION.md](docs/INSTALLATION.md)

### Plugin Install (one command)

| Platform | Command |
|----------|---------|
| **Claude Code** | `/plugin install BandaruDheeraj/N8N-Workflow-Creator` |
| **GitHub Copilot CLI** | `copilot plugin install BandaruDheeraj/N8N-Workflow-Creator` |
| **Cursor** | `/plugin-add n8n-workflow-creator` |

### Skill Install (copy files)

| Platform | Method |
|----------|--------|
| **Codex** | `Fetch and follow instructions from https://raw.githubusercontent.com/BandaruDheeraj/N8N-Workflow-Creator/main/.codex/INSTALL.md` |
| **OpenCode** | `Fetch and follow instructions from https://raw.githubusercontent.com/BandaruDheeraj/N8N-Workflow-Creator/main/.opencode/INSTALL.md` |
| **Copilot CLI (skills)** | `Fetch and follow instructions from https://raw.githubusercontent.com/BandaruDheeraj/N8N-Workflow-Creator/main/.copilot-cli/INSTALL.md` |
| **Copilot Custom Agent** | Copy skills to `.github/agents/` — see [docs/INSTALLATION.md](docs/INSTALLATION.md) |
| **Manual** | `git clone` + copy `skills/` into your agent's skill directory |

## Prerequisites

- **Python 3.8+** with `requests`
- **n8n instance** (Cloud or self-hosted) with API key
- **Playwright MCP** (optional, for browser validation)

## License

MIT — see [LICENSE](LICENSE).

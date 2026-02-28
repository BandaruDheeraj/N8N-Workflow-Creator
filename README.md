# N8N Workflow Creator

Installable skills that teach coding agents to **build, test, and debug n8n workflows via the REST API** — no browser UI needed. Built from battle-tested production workflows.

## What You Get

| Skill | What It Teaches |
|-------|-----------------|
| **n8n-workflow-building** | Create nodes, wire connections, deploy workflows programmatically |
| **n8n-workflow-testing** | Trigger → poll → validate end-to-end, because 200 ≠ success |
| **n8n-workflow-debugging** | Trace data flow, find field mismatches, fix silent data loss |
| **n8n-api-patterns** | Code node HTTP, LLM parallelization, 60s timeout workarounds |
| **n8n-google-sheets** | Append, update, batch ops, cleanup, rate limiting |
| **n8n-email-outreach** | Hunter.io verification, contact ranking, AI cold emails, mail merge |

## Install

### Claude Code
```
/plugin install BandaruDheeraj/N8N-Workflow-Creator
```

### Cursor
```
/plugin-add n8n-workflow-creator
```

### Codex
```
Fetch and follow instructions from https://raw.githubusercontent.com/BandaruDheeraj/N8N-Workflow-Creator/main/.codex/INSTALL.md
```

### OpenCode
```
Fetch and follow instructions from https://raw.githubusercontent.com/BandaruDheeraj/N8N-Workflow-Creator/main/.opencode/INSTALL.md
```

### GitHub Copilot
Copy `skills/*/SKILL.md` into your repo's `.github/agents/` directory.

### GitHub Copilot CLI
```bash
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git
# Copy skills as instruction files
mkdir -p .github/instructions/n8n
cp N8N-Workflow-Creator/skills/*/SKILL.md .github/instructions/n8n/
# Rename to .instructions.md format
cd .github/instructions/n8n
for f in SKILL.md; do true; done
```
Or one-liner:
```bash
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git && mkdir -p .github/instructions/n8n && for d in N8N-Workflow-Creator/skills/*/; do cp "$d/SKILL.md" ".github/instructions/n8n/$(basename $d).instructions.md"; done
```
Copilot CLI auto-loads all `*.instructions.md` files from `.github/instructions/`.

### Manual
```bash
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git
# Symlink skills/ into your agent's skill directory
```

## Prerequisites

- **Python 3.8+** with `requests`
- **n8n instance** (Cloud or self-hosted) with API key
- **Playwright MCP** (optional, for browser validation)

## License

MIT — see [LICENSE](LICENSE).

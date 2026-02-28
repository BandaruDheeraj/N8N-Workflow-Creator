# Installation Guide

## Claude Code (Plugin)

```
/plugin install <owner>/N8N-Workflow-Creator
```

Or via marketplace:
```
/plugin marketplace add <owner>/N8N-Workflow-Creator
/plugin install n8n-workflow-creator
```

## Cursor

```
/plugin-add n8n-workflow-creator
```

## Codex

```
Fetch and follow instructions from https://raw.githubusercontent.com/<owner>/N8N-Workflow-Creator/refs/heads/main/.codex/INSTALL.md
```

## OpenCode

```
Fetch and follow instructions from https://raw.githubusercontent.com/<owner>/N8N-Workflow-Creator/refs/heads/main/.opencode/INSTALL.md
```

## GitHub Copilot

Copy the skills you need into your repo's `.github/agents/` directory:

```bash
# Clone
git clone https://github.com/<owner>/N8N-Workflow-Creator.git

# Copy all skills as a single agent file
cp N8N-Workflow-Creator/skills/*/SKILL.md .github/agents/n8n-workflow-creator.agent.md
```

Or reference individual skills in your custom agent configuration.

## Manual Installation

```bash
git clone https://github.com/<owner>/N8N-Workflow-Creator.git
```

Then symlink or copy the `skills/` directory to your agent platform's skill location:

| Platform | Skill Directory |
|----------|----------------|
| Claude Code | `~/.claude/skills/` |
| Codex | `~/.agents/skills/` |
| OpenCode | `~/.config/opencode/skills/` |

## Prerequisites

Your agent environment needs:
- **Python 3.8+** with `requests` library (`pip install requests`)
- **n8n instance** (Cloud or self-hosted) with API key enabled
- **Playwright MCP** (optional) for browser-based validation

## Verify Installation

Start a new session and try one of these prompts:
- "Build an n8n webhook workflow that writes to Google Sheets"
- "Debug why my n8n workflow produces 0 items at the Hunter Search node"
- "Test my n8n workflow end-to-end and verify the sheet was updated"

The agent should automatically invoke the relevant n8n skill.

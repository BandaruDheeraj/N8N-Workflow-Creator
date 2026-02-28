# Installation Guide

## Claude Code (Plugin)

```
/plugin install BandaruDheeraj/N8N-Workflow-Creator
```

Or via marketplace:
```
/plugin marketplace add BandaruDheeraj/N8N-Workflow-Creator
/plugin install n8n-workflow-creator
```

## Cursor

```
/plugin-add n8n-workflow-creator
```

## Codex

```
Fetch and follow instructions from https://raw.githubusercontent.com/BandaruDheeraj/N8N-Workflow-Creator/refs/heads/main/.codex/INSTALL.md
```

## OpenCode

```
Fetch and follow instructions from https://raw.githubusercontent.com/BandaruDheeraj/N8N-Workflow-Creator/refs/heads/main/.opencode/INSTALL.md
```

## GitHub Copilot (Custom Agent)

Copy the skills you need into your repo's `.github/agents/` directory:

```bash
# Clone
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git

# Copy all skills as a single agent file
cp N8N-Workflow-Creator/skills/*/SKILL.md .github/agents/n8n-workflow-creator.agent.md
```

Or reference individual skills in your custom agent configuration.

## GitHub Copilot CLI

Copilot CLI auto-loads `*.instructions.md` files from `.github/instructions/` in your repo. Copy each skill as a separate instruction file:

```bash
# Clone
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git

# Copy skills as instruction files
mkdir -p .github/instructions/n8n
for d in N8N-Workflow-Creator/skills/*/; do
  cp "$d/SKILL.md" ".github/instructions/n8n/$(basename $d).instructions.md"
done
```

This creates files like `.github/instructions/n8n/n8n-workflow-building.instructions.md` that Copilot CLI picks up automatically.

For global access (all repos), place instruction files in `$HOME/.copilot/copilot-instructions.md` or set the `COPILOT_CUSTOM_INSTRUCTIONS_DIRS` env var. Verify with `/skills` inside a Copilot CLI session.

## Manual Installation

```bash
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git
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

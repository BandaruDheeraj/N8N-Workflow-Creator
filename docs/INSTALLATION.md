# Installation Guide

All methods below install the same 7 skills. Pick the one that matches your tool.

---

## Plugin Install (Recommended)

One-command install — the plugin manifest handles discovery and registration automatically.

### Claude Code

```
/plugin install BandaruDheeraj/N8N-Workflow-Creator
```

Claude Code reads `.claude-plugin/plugin.json` and registers all 7 skills. No manual file copying needed.

### GitHub Copilot CLI

```
copilot plugin install BandaruDheeraj/N8N-Workflow-Creator
```

Copilot CLI reads `plugin.json` at the repo root and registers all 7 skills. Verify with `copilot plugin list`.

### Cursor

```
/plugin-add n8n-workflow-creator
```

---

## Skill Install (Copy Files)

For platforms without plugin support, or if you prefer manual control.

### Codex

Paste this prompt into a Codex session:

```
Fetch and follow instructions from https://raw.githubusercontent.com/BandaruDheeraj/N8N-Workflow-Creator/refs/heads/main/.codex/INSTALL.md
```

This installs skills as `.codex/skills/<name>/SKILL.md` folders in your project. Codex auto-discovers them as invokable skills.

### OpenCode

Paste this prompt into an OpenCode session:

```
Fetch and follow instructions from https://raw.githubusercontent.com/BandaruDheeraj/N8N-Workflow-Creator/refs/heads/main/.opencode/INSTALL.md
```

This installs skills as `.opencode/skills/<name>/SKILL.md` folders in your project. OpenCode auto-discovers them as invokable skills.

### GitHub Copilot (Custom Agent)

Skills are loaded as `.agent.md` files in your repo's `.github/agents/` directory.

#### macOS / Linux

```bash
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git
mkdir -p .github/agents
for d in N8N-Workflow-Creator/skills/*/; do
  cp "$d/SKILL.md" ".github/agents/$(basename $d).agent.md"
done
```

#### Windows (PowerShell)

```powershell
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git
New-Item -ItemType Directory -Force -Path .github\agents | Out-Null
Get-ChildItem -Directory N8N-Workflow-Creator\skills | ForEach-Object {
  Copy-Item "$($_.FullName)\SKILL.md" ".github\agents\$($_.Name).agent.md"
}
```

This creates files like `.github/agents/n8n-workflow-building.agent.md`.

### GitHub Copilot CLI (Skills Only)

If you prefer skills over the plugin install:

```
Fetch and follow instructions from https://raw.githubusercontent.com/BandaruDheeraj/N8N-Workflow-Creator/main/.copilot-cli/INSTALL.md
```

This installs skills as `.github/skills/<name>/SKILL.md` folders (project-level) or `~/.copilot/skills/<name>/SKILL.md` (personal/global). Verify with `/skills list` in a Copilot CLI session.

---

## Manual Installation

Clone and then symlink or copy `skills/` into your agent platform's expected location.

### macOS / Linux

```bash
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git
# Example: symlink into a custom skill directory
ln -s "$(pwd)/N8N-Workflow-Creator/skills" /path/to/your/agent/skills/n8n
```

### Windows (PowerShell)

```powershell
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git
# Example: copy into a custom skill directory
Copy-Item -Recurse N8N-Workflow-Creator\skills C:\path\to\your\agent\skills\n8n
```

**Common skill directories by platform:**

| Platform | Project-level | Global |
|----------|---------------|--------|
| Claude Code | `.claude/skills/` | `~/.claude/skills/` |
| Codex | `.codex/skills/` | `~/.codex/skills/` |
| OpenCode | `.opencode/skills/` | `~/.config/opencode/skills/` |
| Copilot CLI | `.github/skills/` | `~/.copilot/skills/` |

---

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

## Cleanup

After copying the skill files, you can remove the cloned repo:

**macOS / Linux:**
```bash
rm -rf N8N-Workflow-Creator
```

**Windows (PowerShell):**
```powershell
Remove-Item -Recurse -Force N8N-Workflow-Creator
```

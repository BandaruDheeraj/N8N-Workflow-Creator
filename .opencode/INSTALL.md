# Install N8N Workflow Creator Skills for OpenCode

OpenCode loads skills from `.opencode/skills/<name>/SKILL.md` (project-level) or `~/.config/opencode/skills/<name>/SKILL.md` (global).

## Project Install (Recommended)

```bash
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git /tmp/n8n-skills
mkdir -p .opencode/skills
cp -r /tmp/n8n-skills/skills/* .opencode/skills/
rm -rf /tmp/n8n-skills
```

## Global Install

```bash
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git ~/N8N-Workflow-Creator
mkdir -p ~/.config/opencode/skills
for d in ~/N8N-Workflow-Creator/skills/*/; do
  ln -s "$d" ~/.config/opencode/skills/$(basename "$d")
done
```

Update global skills with `cd ~/N8N-Workflow-Creator && git pull` — symlinks update instantly.

## Verify

Confirm these 6 skill folders exist:

- `.opencode/skills/n8n-workflow-building/SKILL.md` (or `~/.config/opencode/skills/` for global)
- `.opencode/skills/n8n-workflow-testing/SKILL.md`
- `.opencode/skills/n8n-workflow-debugging/SKILL.md`
- `.opencode/skills/n8n-api-patterns/SKILL.md`
- `.opencode/skills/n8n-google-sheets/SKILL.md`
- `.opencode/skills/n8n-email-outreach/SKILL.md`

Start a new OpenCode session — skills should be available when relevant tasks are requested.

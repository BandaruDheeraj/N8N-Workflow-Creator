# Install N8N Workflow Creator Skills for Codex

Codex loads skills from `.codex/skills/<name>/SKILL.md` (project-level) or `~/.codex/skills/<name>/SKILL.md` (global).

## Project Install (Recommended)

```bash
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git /tmp/n8n-skills
mkdir -p .codex/skills
cp -r /tmp/n8n-skills/skills/* .codex/skills/
rm -rf /tmp/n8n-skills
```

## Global Install

```bash
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git ~/N8N-Workflow-Creator
mkdir -p ~/.codex/skills
for d in ~/N8N-Workflow-Creator/skills/*/; do
  ln -s "$d" ~/.codex/skills/$(basename "$d")
done
```

Update global skills with `cd ~/N8N-Workflow-Creator && git pull` — symlinks update instantly.

## Verify

Confirm these 8 skill folders exist:

- `.codex/skills/n8n-full-pipeline-setup/SKILL.md` (or `~/.codex/skills/` for global)
- `.codex/skills/n8n-workflow-building/SKILL.md`
- `.codex/skills/n8n-workflow-testing/SKILL.md`
- `.codex/skills/n8n-workflow-debugging/SKILL.md`
- `.codex/skills/n8n-api-patterns/SKILL.md`
- `.codex/skills/n8n-google-sheets/SKILL.md`
- `.codex/skills/n8n-email-outreach/SKILL.md`
- `.codex/skills/n8n-exa-instagram-discovery/SKILL.md`

Start a new Codex session — skills should be available when relevant tasks are requested.

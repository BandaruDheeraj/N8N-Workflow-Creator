# Install N8N Workflow Creator Skills for Codex

## Quick Install

```bash
# 1. Clone the repository
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git ~/N8N-Workflow-Creator

# 2. Symlink to Codex skills directory
mkdir -p ~/.agents/skills
ln -s ~/N8N-Workflow-Creator/skills ~/.agents/skills/n8n-workflow-creator

# 3. Verify
ls ~/.agents/skills/n8n-workflow-creator/
```

## Update

```bash
cd ~/N8N-Workflow-Creator && git pull
```

Skills update instantly via symlink — no restart needed.

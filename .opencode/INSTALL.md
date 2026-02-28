# Install N8N Workflow Creator Skills for OpenCode

## Quick Install

```bash
# 1. Clone the repository
git clone https://github.com/<owner>/N8N-Workflow-Creator.git ~/N8N-Workflow-Creator

# 2. Symlink to OpenCode skills directory
mkdir -p ~/.config/opencode/skills
ln -s ~/N8N-Workflow-Creator/skills ~/.config/opencode/skills/n8n-workflow-creator

# 3. Verify
ls ~/.config/opencode/skills/n8n-workflow-creator/
```

## Update

```bash
cd ~/N8N-Workflow-Creator && git pull
```

Skills update instantly via symlink — no restart needed.

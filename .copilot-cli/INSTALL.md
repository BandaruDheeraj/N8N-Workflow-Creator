# Install N8N Workflow Creator Skills for GitHub Copilot CLI

Copilot CLI loads skills from `skills/<name>/SKILL.md` in the current repo. Each skill folder contains a `SKILL.md` with frontmatter (`name` and `description`) that Copilot CLI uses to register them as invokable skills.

> **Important:** Do NOT install these as `*.instructions.md` in `.github/instructions/` — that makes them always-on instructions, not on-demand skills.

## Steps

1. Clone the skill repository into a temporary directory
2. Create `skills/` in the current repo (if it doesn't exist)
3. Copy each skill folder (`skills/<name>/SKILL.md`) into the current repo's `skills/` directory
4. Remove the cloned temporary directory

## Install Commands

Detect the OS and run the appropriate commands:

### macOS / Linux (bash)

```bash
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git /tmp/n8n-skills
cp -r /tmp/n8n-skills/skills/* skills/ 2>/dev/null || mkdir -p skills && cp -r /tmp/n8n-skills/skills/* skills/
rm -rf /tmp/n8n-skills
```

### Windows (PowerShell)

```powershell
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git "$env:TEMP\n8n-skills"
if (-not (Test-Path skills)) { New-Item -ItemType Directory -Path skills | Out-Null }
Copy-Item -Recurse -Force "$env:TEMP\n8n-skills\skills\*" skills\
Remove-Item -Recurse -Force "$env:TEMP\n8n-skills"
```

## Verify

After running the install, confirm these 6 skill folders exist in `skills/`:

- `skills/n8n-workflow-building/SKILL.md`
- `skills/n8n-workflow-testing/SKILL.md`
- `skills/n8n-workflow-debugging/SKILL.md`
- `skills/n8n-api-patterns/SKILL.md`
- `skills/n8n-google-sheets/SKILL.md`
- `skills/n8n-email-outreach/SKILL.md`

List the directory to confirm all folders were created successfully. Then start a new Copilot CLI session — the skills should appear when relevant tasks are requested.

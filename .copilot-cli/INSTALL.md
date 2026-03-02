# Install N8N Workflow Creator for GitHub Copilot CLI

## Plugin Install (Recommended)

```
copilot plugin install BandaruDheeraj/N8N-Workflow-Creator
```

This reads `plugin.json` and registers all 8 skills automatically. Verify with `copilot plugin list`.

## Skill Install (Manual)

Copilot CLI discovers project skills from `.github/skills/<name>/SKILL.md` and personal skills from `~/.copilot/skills/<name>/SKILL.md`.

## Project Install (Recommended)

Installs skills into the current repo so they're available when working in this project.

### macOS / Linux (bash)

```bash
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git /tmp/n8n-skills
mkdir -p .github/skills
cp -r /tmp/n8n-skills/skills/* .github/skills/
rm -rf /tmp/n8n-skills
```

### Windows (PowerShell)

```powershell
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git "$env:TEMP\n8n-skills"
if (-not (Test-Path .github\skills)) { New-Item -ItemType Directory -Force -Path .github\skills | Out-Null }
Copy-Item -Recurse -Force "$env:TEMP\n8n-skills\skills\*" .github\skills\
Remove-Item -Recurse -Force "$env:TEMP\n8n-skills"
```

## Personal Install (All Projects)

Installs skills globally so they're available in every Copilot CLI session.

### macOS / Linux (bash)

```bash
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git /tmp/n8n-skills
mkdir -p ~/.copilot/skills
cp -r /tmp/n8n-skills/skills/* ~/.copilot/skills/
rm -rf /tmp/n8n-skills
```

### Windows (PowerShell)

```powershell
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git "$env:TEMP\n8n-skills"
$dest = Join-Path $env:USERPROFILE ".copilot\skills"
if (-not (Test-Path $dest)) { New-Item -ItemType Directory -Force -Path $dest | Out-Null }
Copy-Item -Recurse -Force "$env:TEMP\n8n-skills\skills\*" $dest\
Remove-Item -Recurse -Force "$env:TEMP\n8n-skills"
```

## Verify

After installing, run `/skills list` in a Copilot CLI session. You should see all 8 skills:

- `n8n-full-pipeline-setup`
- `n8n-workflow-building`
- `n8n-workflow-testing`
- `n8n-workflow-debugging`
- `n8n-api-patterns`
- `n8n-google-sheets`
- `n8n-email-outreach`
- `n8n-exa-instagram-discovery`

You can also verify the files exist:

**Project install:**
- `.github/skills/n8n-full-pipeline-setup/SKILL.md`
- `.github/skills/n8n-workflow-building/SKILL.md`
- `.github/skills/n8n-workflow-testing/SKILL.md`
- `.github/skills/n8n-workflow-debugging/SKILL.md`
- `.github/skills/n8n-api-patterns/SKILL.md`
- `.github/skills/n8n-google-sheets/SKILL.md`
- `.github/skills/n8n-email-outreach/SKILL.md`
- `.github/skills/n8n-exa-instagram-discovery/SKILL.md`

**Personal install:**
- `~/.copilot/skills/n8n-workflow-building/SKILL.md` (same names)

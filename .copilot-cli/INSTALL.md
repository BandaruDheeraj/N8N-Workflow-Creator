# Install N8N Workflow Creator Skills for GitHub Copilot CLI

Copilot CLI loads `*.instructions.md` files from `.github/instructions/` in the current repo.

## Steps

1. Clone the skill repository into a temporary directory
2. Create `.github/instructions/n8n/` in the current repo
3. Copy each `skills/*/SKILL.md` as `<skill-name>.instructions.md`
4. Remove the cloned temporary directory

## Install Commands

Detect the OS and run the appropriate commands:

### macOS / Linux (bash)

```bash
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git /tmp/n8n-skills
mkdir -p .github/instructions/n8n
for d in /tmp/n8n-skills/skills/*/; do
  cp "$d/SKILL.md" ".github/instructions/n8n/$(basename $d).instructions.md"
done
rm -rf /tmp/n8n-skills
```

### Windows (PowerShell)

```powershell
git clone https://github.com/BandaruDheeraj/N8N-Workflow-Creator.git "$env:TEMP\n8n-skills"
New-Item -ItemType Directory -Force -Path .github\instructions\n8n | Out-Null
Get-ChildItem -Directory "$env:TEMP\n8n-skills\skills" | ForEach-Object {
  Copy-Item "$($_.FullName)\SKILL.md" ".github\instructions\n8n\$($_.Name).instructions.md"
}
Remove-Item -Recurse -Force "$env:TEMP\n8n-skills"
```

## Verify

After running the install, confirm these 6 files exist in `.github/instructions/n8n/`:

- `n8n-workflow-building.instructions.md`
- `n8n-workflow-testing.instructions.md`
- `n8n-workflow-debugging.instructions.md`
- `n8n-api-patterns.instructions.md`
- `n8n-google-sheets.instructions.md`
- `n8n-email-outreach.instructions.md`

List the directory to confirm all files were created successfully.

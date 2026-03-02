# N8N Workflow Creator

Installable skills that teach coding agents to **build, test, and debug n8n workflows via the REST API** — no browser UI needed. Built from battle-tested production workflows.

## How It Works

You describe what you want in plain English. The skills auto-activate based on your prompt — no manual invocation needed.

### 1. Describe your workflow

> *"Build me an n8n workflow that takes a list of company domains, finds contacts via Hunter.io, scores them, generates personalized cold emails, and writes everything to a Google Sheet for mail merge."*

### 2. Skills chain automatically

| Phase | Skill That Fires | What It Does |
|-------|-------------------|--------------|
| **Environment check** | `n8n-full-pipeline-setup` | Validates n8n is reachable, discovers credentials, plans the node pipeline |
| **Build** | `n8n-workflow-building` | Creates nodes, wires connections, deploys via REST API |
| **Domain logic** | `n8n-api-patterns` | Handles Code node HTTP calls, LLM parallelization, 60s timeout batching |
| **Email gen** | `n8n-email-outreach` | Hunter.io verification, contact ranking, AI email humanization |
| **Sheet output** | `n8n-google-sheets` | Appends rows, creates tabs, handles column alignment |
| **Test** | `n8n-workflow-testing` | Triggers execution, polls status, validates node-by-node output |
| **Debug** | `n8n-workflow-debugging` | Only fires if something fails — traces data flow, finds the break |

### 3. Verify acceptance criteria

Ask the agent to confirm the output:

> *"Verify the data reached the Google Sheet and show me what's in the output tab."*

### 4. Iterate if needed

> *"The Hunter Search node is returning 0 items — debug it."*

The debugging skill traces the data flow and finds the field mismatch.

### Tips

- **Be specific about acceptance criteria upfront** — e.g. *"...and I need at least 3 verified contacts per company written to the sheet"* — so the testing skill knows what to validate.
- **You can invoke a skill explicitly** if auto-detection misses — just mention the topic (*"debug why node X produces 0 items"* → debugging skill, *"test the workflow end-to-end"* → testing skill).
- **The full-pipeline skill is the orchestrator** — say *"build me a workflow from scratch"* and it coordinates all the other skills in the right order.
- **You don't need to run skills one-at-a-time** — describe the whole goal and let the agent plan the sequence.

---

## What You Get

| Skill | What It Teaches |
|-------|-----------------|
| **n8n-full-pipeline-setup** | End-to-end setup: environment validation → credentials → build → deploy → test → verify |
| **n8n-workflow-building** | Create nodes, wire connections, deploy workflows programmatically |
| **n8n-workflow-testing** | Trigger → poll → validate end-to-end, because 200 ≠ success |
| **n8n-workflow-debugging** | Trace data flow, find field mismatches, fix silent data loss |
| **n8n-api-patterns** | Code node HTTP, LLM parallelization, 60s timeout workarounds |
| **n8n-google-sheets** | Append, update, batch ops, cleanup, rate limiting |
| **n8n-email-outreach** | Hunter.io verification, contact ranking, AI cold emails, mail merge |

## Install

> **Full install guide with all options:** [docs/INSTALLATION.md](docs/INSTALLATION.md)

### Plugin Install (one command)

| Platform | Command |
|----------|---------|
| **Claude Code** | `/plugin install BandaruDheeraj/N8N-Workflow-Creator` |
| **GitHub Copilot CLI** | `copilot plugin install BandaruDheeraj/N8N-Workflow-Creator` |
| **Cursor** | `/plugin-add n8n-workflow-creator` |

### Skill Install (copy files)

| Platform | Method |
|----------|--------|
| **Codex** | `Fetch and follow instructions from https://raw.githubusercontent.com/BandaruDheeraj/N8N-Workflow-Creator/main/.codex/INSTALL.md` |
| **OpenCode** | `Fetch and follow instructions from https://raw.githubusercontent.com/BandaruDheeraj/N8N-Workflow-Creator/main/.opencode/INSTALL.md` |
| **Copilot CLI (skills)** | `Fetch and follow instructions from https://raw.githubusercontent.com/BandaruDheeraj/N8N-Workflow-Creator/main/.copilot-cli/INSTALL.md` |
| **Copilot Custom Agent** | Copy skills to `.github/agents/` — see [docs/INSTALLATION.md](docs/INSTALLATION.md) |
| **Manual** | `git clone` + copy `skills/` into your agent's skill directory |

## Prerequisites

- **Python 3.8+** with `requests`
- **n8n instance** (Cloud or self-hosted) with API key
- **Playwright MCP** (optional, for browser validation)

## 🤝 Use with czlonkowski/n8n-skills

This project is designed to work **alongside** [czlonkowski/n8n-skills](https://github.com/czlonkowski/n8n-skills) — the popular n8n skill set for Claude Code. They're complementary, not competing.

**They teach the agent how to speak n8n** — expression syntax, node configuration, validation errors, MCP tool usage.
**We teach the agent how to get things done with n8n** — build a pipeline, deploy it, test it end-to-end, debug it when data disappears, get results into Sheets, send outreach emails.

| | czlonkowski/n8n-skills | N8N-Workflow-Creator |
|---|---|---|
| **Focus** | Configure n8n correctly | Operate n8n end-to-end |
| **Covers** | Expressions, node config, validation, JS/Python code | Pipeline setup, testing, debugging, Sheets, email outreach |
| **Approach** | MCP tool calls via n8n-mcp server | Direct REST API calls (no MCP needed) |
| **Platforms** | Claude Code | 7 platforms (Claude, Cursor, Codex, OpenCode, Copilot) |

### Install both

```
# In Claude Code — install both plugins
/plugin install czlonkowski/n8n-skills
/plugin install BandaruDheeraj/N8N-Workflow-Creator
```

With both installed, your agent knows *how to configure nodes correctly* (czlonkowski) **and** *how to build, deploy, test, and debug complete workflows* (this project). The combination covers the full n8n development lifecycle — from writing a correct expression to verifying data landed in your Google Sheet.

## License

MIT — see [LICENSE](LICENSE).

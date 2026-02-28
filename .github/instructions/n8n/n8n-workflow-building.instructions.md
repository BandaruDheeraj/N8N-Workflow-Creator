---
name: n8n-workflow-building
description: Use when creating n8n workflows programmatically via REST API, wiring nodes and connections, deploying workflow updates, or designing workflow architecture without the browser UI.
---

# n8n Workflow Building via REST API

## Overview

Build and deploy n8n workflows entirely through the REST API using Python + `requests`. No browser UI needed — create nodes, wire connections, set parameters, and deploy in one script.

## When to Use

- Creating a new n8n workflow from scratch
- Adding/removing/rewiring nodes in an existing workflow
- Deploying workflow changes programmatically
- Designing multi-node pipelines (lead gen, data processing, outreach)

## Core Pattern: Read → Modify → Write

```python
import requests, json

API_KEY = "<your-n8n-api-key>"
BASE = "https://<instance>.app.n8n.cloud"
h = {"X-N8N-API-KEY": API_KEY, "Content-Type": "application/json"}
WID = "<workflow-id>"

# 1. Fetch current workflow
wf = requests.get(f"{BASE}/api/v1/workflows/{WID}", headers=h).json()

# 2. Modify nodes/connections
for node in wf["nodes"]:
    if node["name"] == "My Code Node":
        node["parameters"]["jsCode"] = "return $input.all();"

# 3. Write back (ONLY these fields are accepted)
payload = {
    "name": wf["name"],
    "nodes": wf["nodes"],
    "connections": wf["connections"],
    "settings": wf["settings"],
    "staticData": wf["staticData"],
}
# Do NOT include: tags, id, createdAt, updatedAt, active, meta, pinData
resp = requests.put(f"{BASE}/api/v1/workflows/{WID}", headers=h, json=payload)
```

## Connection Format

```python
# Source node output 0 → Target node input 0
wf["connections"]["Source Node"] = {
    "main": [
        [{"node": "Target Node", "type": "main", "index": 0}],  # output 0
        []  # output 1 (empty = disconnected)
    ]
}

# Fan-out: one node feeds two downstream nodes
wf["connections"]["Format Results"] = {
    "main": [[
        {"node": "Add to Sheets", "type": "main", "index": 0},
        {"node": "Select Priority Contacts", "type": "main", "index": 0}
    ]]
}

# AI sub-nodes use ai_languageModel, NOT main
wf["connections"]["OpenRouter Model"] = {
    "ai_languageModel": [[{
        "node": "AI Chain Node",
        "type": "ai_languageModel",
        "index": 0
    }]]
}
```

## Activation Cycling (Required After Updates)

After ANY workflow update, you MUST cycle activation for changes to take effect:

```python
requests.post(f"{BASE}/api/v1/workflows/{WID}/deactivate", headers=h)
time.sleep(3)
requests.post(f"{BASE}/api/v1/workflows/{WID}/activate", headers=h)
time.sleep(5)  # webhook registration needs time to propagate
```

## Workflow Architecture Templates

### Single-destination pipeline
```
Trigger → Process → Enrich → Write to Sheet
```

### Dual-destination with priority selection
```
Trigger ──┬→ Brand List ──────────→ Merge (input 1, append)
          ├→ AI Generate → Parse →  Merge (input 0)
          └→ Read Existing (no output wire — accessed via $() ref)

Merge → Dedup → Hunter Search → Format Results ─┬→ Raw Sheet
                                                 └→ Select Priority → AI Emails → Priority Sheet
```

### Key: Read node without output wire
Trigger a node so it runs, but don't wire its output. Access data via `$('Read Existing Leads').all()` in downstream Code nodes. This avoids Merge node complications.

## Merge Node Configuration

**Always** set mode explicitly. Empty `parameters: {}` causes undefined behavior.

```python
merge_node["parameters"]["mode"] = "append"  # concatenates both inputs
```

**Never use `combineAll`** — it creates a Cartesian product (10 × 10 = 100 items).

## Node Creation Template

```python
new_node = {
    "parameters": { ... },
    "type": "n8n-nodes-base.code",       # or .googleSheets, .hunter, etc.
    "typeVersion": 2,                      # check n8n docs for current version
    "position": [800, 300],                # [x, y] canvas coordinates
    "id": str(uuid.uuid4()),
    "name": "My New Node",                # unique name within the workflow
    "credentials": {}                      # add if node needs auth
}
wf["nodes"].append(new_node)
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Including `id`, `tags`, `active` in PUT body | Only send `name`, `nodes`, `connections`, `settings`, `staticData` |
| Not cycling activation after update | Always deactivate → wait 3s → activate → wait 5s |
| Empty merge parameters `{}` | Explicitly set `"mode": "append"` |
| Using `combineAll` merge mode | Use `append` to concatenate, not Cartesian product |
| Replacing webhook trigger node | Keep original trigger node object intact (same id, webhookId) |

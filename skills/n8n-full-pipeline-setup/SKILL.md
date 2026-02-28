---
name: n8n-full-pipeline-setup
description: Use when setting up a complete n8n workflow from scratch — environment validation, credential discovery, building the full node pipeline, wiring connections, deploying, testing end-to-end, and verifying data reaches its destination.
---

# n8n Full Pipeline Setup

## Overview

Go from zero to a fully operational n8n workflow in one session. This skill orchestrates environment checks, credential discovery, workflow construction, deployment, end-to-end testing, and destination verification — all via the REST API without touching the browser UI.

## When to Use

- Setting up a brand new n8n workflow from scratch
- User says "build me a workflow" or "set up the full pipeline"
- Need to validate the n8n environment before building
- Orchestrating multiple skills (building + testing + sheets) together

## Phase 1: Environment Validation

Before building anything, verify the n8n instance is reachable and the API key works.

```python
import requests, json, time

# Required: user must provide these
API_KEY = "<n8n-api-key>"
BASE = "https://<instance>.app.n8n.cloud"
h = {"X-N8N-API-KEY": API_KEY, "Content-Type": "application/json"}

# Test connectivity
try:
    resp = requests.get(f"{BASE}/api/v1/workflows?limit=1", headers=h, timeout=10)
    resp.raise_for_status()
    print(f"✅ Connected to n8n at {BASE}")
    print(f"   Found {len(resp.json().get('data', []))} workflows")
except requests.exceptions.ConnectionError:
    print(f"❌ Cannot reach {BASE} — check the URL")
except requests.exceptions.HTTPError as e:
    if e.response.status_code == 401:
        print("❌ API key is invalid — check X-N8N-API-KEY")
    else:
        print(f"❌ HTTP {e.response.status_code}: {e.response.text[:200]}")
```

### What to collect from the user

| Item | Example | Required |
|------|---------|----------|
| n8n instance URL | `https://myteam.app.n8n.cloud` | ✅ |
| n8n API key | `n8n_api_...` | ✅ |
| Google Sheet URL | `https://docs.google.com/spreadsheets/d/...` | For sheet workflows |
| Sheet tab name or GID | `High Priority Email` or `1863718260` | For sheet workflows |
| Hunter.io API key | `hunter_...` | For lead gen workflows |
| OpenRouter API key | `sk-or-...` | For AI-powered workflows |

## Phase 2: Credential Discovery

Discover what credentials are already configured in n8n — don't ask users for credential IDs they don't know.

```python
creds = requests.get(f"{BASE}/api/v1/credentials", headers=h).json()

cred_map = {}
for c in creds["data"]:
    cred_map[c["type"]] = {"id": c["id"], "name": c["name"]}
    print(f"  ✅ {c['name']} ({c['type']}) — ID: {c['id']}")

# Check for required credentials
required = {
    "googleSheetsOAuth2Api": "Google Sheets",
    "hunterApi": "Hunter.io",
}
for cred_type, label in required.items():
    if cred_type in cred_map:
        print(f"  ✅ {label}: found ({cred_map[cred_type]['name']})")
    else:
        print(f"  ❌ {label}: not configured — add it in n8n Settings → Credentials")
```

### Building credential references for nodes

```python
def cred_ref(cred_type):
    """Get credential reference for a node, or None if not available."""
    c = cred_map.get(cred_type)
    if c:
        return {cred_type: {"id": c["id"], "name": c["name"]}}
    return {}
```

## Phase 3: Build the Workflow

### Option A: Create a new workflow

```python
import uuid

new_wf = requests.post(f"{BASE}/api/v1/workflows", headers=h, json={
    "name": "Lead Gen Pipeline",
    "nodes": [],
    "connections": {},
    "settings": {"executionOrder": "v1"},
}).json()
WID = new_wf["id"]
print(f"✅ Created workflow {WID}: {new_wf['name']}")
```

### Option B: Update an existing workflow

```python
WID = "<existing-workflow-id>"
wf = requests.get(f"{BASE}/api/v1/workflows/{WID}", headers=h).json()
print(f"✅ Loaded workflow {WID}: {wf['name']} ({len(wf['nodes'])} nodes)")
```

### Standard Lead Gen Pipeline

This builds the most common pipeline: Webhook Trigger → AI Brand Discovery → Dedup → Hunter Search → Format Results → Google Sheets. The webhook trigger accepts JSON via simple POST, making it easy to run programmatically.

```python
SHEET_URL = "https://docs.google.com/spreadsheets/d/<doc-id>"
SHEET_CRED = cred_ref("googleSheetsOAuth2Api")
HUNTER_CRED = cred_ref("hunterApi")

WEBHOOK_PATH = "lead-gen"  # trigger via POST /webhook/lead-gen

nodes = [
    # 1. Webhook Trigger (simple JSON POST — no multipart needed)
    {
        "parameters": {"path": WEBHOOK_PATH, "httpMethod": "POST", "options": {}},
        "type": "n8n-nodes-base.webhook",
        "typeVersion": 2,
        "position": [0, 300],
        "id": str(uuid.uuid4()),
        "name": "Webhook Trigger",
        "webhookId": str(uuid.uuid4()),
    },
    # 2. AI Generate Brands (Code node calling OpenRouter)
    {
        "parameters": {
            "jsCode": """
const count = $input.first().json.count || 10;

const resp = await this.helpers.httpRequest({
    method: 'POST',
    url: 'https://openrouter.ai/api/v1/chat/completions',
    headers: {
        'Authorization': 'Bearer <OPENROUTER_KEY>',
        'Content-Type': 'application/json'
    },
    body: JSON.stringify({
        model: 'anthropic/claude-sonnet-4.6',
        messages: [{
            role: 'user',
            content: `Generate ${count} companies as JSON array: [{"name": "...", "domain": "...", "category": "..."}]`
        }],
        max_tokens: 2000
    })
});

const data = typeof resp === 'string' ? JSON.parse(resp) : resp;
const text = data.choices[0].message.content;
const match = text.match(/\\[[\\s\\S]*\\]/);
const brands = JSON.parse(match[0]);
return brands.map(b => ({ json: b }));
"""
        },
        "type": "n8n-nodes-base.code",
        "typeVersion": 2,
        "position": [300, 300],
        "id": str(uuid.uuid4()),
        "name": "AI Generate Brands",
    },
    # 3. Remove Duplicates
    {
        "parameters": {
            "options": {"removeOtherFields": False},
            "compare": "selectedFields",
            "fieldsToCompare": {"fields": [{"fieldName": "domain"}]}
        },
        "type": "n8n-nodes-base.removeDuplicates",
        "typeVersion": 1,
        "position": [550, 300],
        "id": str(uuid.uuid4()),
        "name": "Remove Duplicates",
    },
    # 4. Hunter Domain Search
    {
        "parameters": {
            "domain": "={{ $json.domain }}",
            "returnAll": False,
            "limit": 5
        },
        "type": "n8n-nodes-base.hunter",
        "typeVersion": 1,
        "position": [800, 300],
        "id": str(uuid.uuid4()),
        "name": "Hunter Search",
        "credentials": HUNTER_CRED,
        "onError": "continueRegularOutput",
    },
    # 5. Format Results (Code node)
    {
        "parameters": {
            "jsCode": """
const results = [];
for (const item of $input.all()) {
    const data = item.json;
    if (!data.emails || !data.emails.length) continue;
    for (const contact of data.emails) {
        const status = (contact.verification && contact.verification.status) || 'unknown';
        if (!['valid', 'accept_all'].includes(status)) continue;
        results.push({
            json: {
                brand_name: data.domain || '',
                brand_domain: data.domain || '',
                email: contact.value || '',
                first_name: contact.first_name || '',
                last_name: contact.last_name || '',
                position: contact.position || '',
                department: contact.department || '',
                confidence: contact.confidence || 0,
                linkedin: contact.linkedin || '',
                verification_status: status,
            }
        });
    }
}
return results.length > 0 ? results : [{ json: { _empty: true } }];
"""
        },
        "type": "n8n-nodes-base.code",
        "typeVersion": 2,
        "position": [1050, 300],
        "id": str(uuid.uuid4()),
        "name": "Format Results",
    },
    # 6. Google Sheets — Append
    {
        "parameters": {
            "operation": "append",
            "documentId": {"mode": "url", "value": SHEET_URL},
            "sheetName": {"mode": "name", "value": "Sheet1"},
            "columns": {"mappingMode": "autoMapInputData", "value": {}},
            "options": {}
        },
        "type": "n8n-nodes-base.googleSheets",
        "typeVersion": 4.5,
        "position": [1300, 300],
        "id": str(uuid.uuid4()),
        "name": "Add to Sheet",
        "credentials": SHEET_CRED,
    },
]

# Wire connections: linear pipeline
connections = {
    "Webhook Trigger":    {"main": [[{"node": "AI Generate Brands", "type": "main", "index": 0}]]},
    "AI Generate Brands": {"main": [[{"node": "Remove Duplicates", "type": "main", "index": 0}]]},
    "Remove Duplicates":  {"main": [[{"node": "Hunter Search", "type": "main", "index": 0}]]},
    "Hunter Search":      {"main": [[{"node": "Format Results", "type": "main", "index": 0}]]},
    "Format Results":     {"main": [[{"node": "Add to Sheet", "type": "main", "index": 0}]]},
}
```

### Deploy the workflow

```python
payload = {
    "name": wf.get("name", "Lead Gen Pipeline"),
    "nodes": nodes,
    "connections": connections,
    "settings": {"executionOrder": "v1"},
    "staticData": wf.get("staticData", None),
}
resp = requests.put(f"{BASE}/api/v1/workflows/{WID}", headers=h, json=payload)
resp.raise_for_status()
print(f"✅ Deployed {len(nodes)} nodes with {len(connections)} connections")
```

## Phase 4: Activate & Test

### Activation cycling

```python
requests.post(f"{BASE}/api/v1/workflows/{WID}/deactivate", headers=h)
time.sleep(3)
requests.post(f"{BASE}/api/v1/workflows/{WID}/activate", headers=h)
time.sleep(5)
print("✅ Workflow activated")
```

### Record baseline and trigger

```python
# Baseline: record existing execution IDs
known = set(str(e["id"]) for e in requests.get(
    f"{BASE}/api/v1/executions?workflowId={WID}&limit=10", headers=h
).json()["data"])

# Trigger the webhook with a simple JSON POST
resp = requests.post(
    f"{BASE}/webhook/{WEBHOOK_PATH}",
    json={"count": 5}
)
print(f"Trigger response: {resp.status_code}")
# ⚠️ 200 does NOT mean success — must poll executions
```

### Poll for new execution

```python
new_exec_id = None
for attempt in range(40):
    time.sleep(3)
    execs = requests.get(
        f"{BASE}/api/v1/executions?workflowId={WID}&limit=5", headers=h
    ).json()["data"]
    for e in execs:
        if str(e["id"]) not in known:
            new_exec_id = e["id"]
            break
    if new_exec_id:
        print(f"✅ New execution: {new_exec_id} (found after {(attempt+1)*3}s)")
        break

if not new_exec_id:
    print("❌ No new execution after 120s — check webhook registration")
    print("   Try: deactivate → 3s → activate → 5s → trigger again")
```

### Wait for completion and validate

```python
for _ in range(60):
    time.sleep(5)
    ed = requests.get(
        f"{BASE}/api/v1/executions/{new_exec_id}?includeData=true", headers=h
    ).json()
    if ed["status"] in ("success", "error"):
        break

print(f"\n{'='*60}")
print(f"Execution {new_exec_id}: {ed['status'].upper()}")
print(f"{'='*60}")

# Node-by-node results
rd = ed["data"]["resultData"]["runData"]
all_ok = True
for name, runs in rd.items():
    for run in runs:
        items = sum(len(o) for o in (run.get("data", {}).get("main") or []) if o)
        status = run["executionStatus"]
        err = ""
        if run.get("error"):
            err = f" — {run['error'].get('message', '')[:80]}"
            all_ok = False
        icon = "✅" if status == "success" and items > 0 else "⚠️" if items == 0 else "❌"
        print(f"  {icon} {name}: {items} items ({status}){err}")

if all_ok and ed["status"] == "success":
    print(f"\n🎉 Pipeline is fully operational!")
else:
    print(f"\n⚠️ Issues detected — see node details above")
```

## Phase 5: Verify Destination

### Check Google Sheets received data

```python
# Use the execution data to verify what was sent to the sheet
sheet_node = rd.get("Add to Sheet", [{}])[0]
sheet_items = sum(len(o) for o in (sheet_node.get("data", {}).get("main") or []) if o)
if sheet_items > 0:
    # Show sample of what was written
    sample = sheet_node["data"]["main"][0][:3]
    print(f"\n✅ {sheet_items} rows sent to Google Sheets")
    for item in sample:
        j = item["json"]
        print(f"   {j.get('brand_name', '?')} — {j.get('email', '?')} ({j.get('verification_status', '?')})")
else:
    print("\n❌ 0 rows sent to sheet — check Format Results node output")
```

### Deactivate after testing

```python
requests.post(f"{BASE}/api/v1/workflows/{WID}/deactivate", headers=h)
print("✅ Workflow deactivated (safe from accidental triggers)")
```

## Pipeline Variations

### Form trigger (for browser/human use)

Replace the Webhook Trigger with a Form Trigger for a user-facing form:
```python
form_trigger = {
    "parameters": {
        "formTitle": "Lead Gen",
        "formFields": {"values": [
            {"fieldLabel": "Number of brands", "fieldType": "number", "requiredField": True}
        ]},
        "options": {}
    },
    "type": "n8n-nodes-base.formTrigger",
    "typeVersion": 2.2,
    "position": [0, 300],
    "id": str(uuid.uuid4()),
    "name": "Form Trigger",
    "webhookId": str(uuid.uuid4()),
}
# ⚠️ Form triggers require multipart/form-data — see n8n-workflow-testing skill
# In downstream code: $input.first().json['Number of brands']
```

### Add priority email branch

Fork after Format Results to also generate personalized outreach emails:
```python
# Fan-out from Format Results
connections["Format Results"] = {
    "main": [[
        {"node": "Add to Sheet", "type": "main", "index": 0},
        {"node": "Select Priority Contacts", "type": "main", "index": 0},
    ]]
}
# Then: Select Priority Contacts → AI Generate Emails → Add to Priority Sheet
```

See the **n8n-email-outreach** skill for contact ranking and AI email generation details.

### Add existing lead dedup

Wire a Read Existing Leads node from the trigger (no output wire — access via `$()` reference):
```python
connections["Webhook Trigger"]["main"][0].append(
    {"node": "Read Existing Leads", "type": "main", "index": 0}
)
# In AI Generate Brands code: $('Read Existing Leads').all() for exclusion list
```

## Setup Checklist

Run through this after the pipeline test:

- [ ] n8n instance reachable and API key valid
- [ ] All required credentials found (Sheets, Hunter, etc.)
- [ ] Workflow created and deployed with correct node count
- [ ] Activation cycling completed (deactivate → activate)
- [ ] Trigger fires and new execution appears within 30s
- [ ] Execution completes with status `success`
- [ ] Every node shows `> 0` items (no silent drops)
- [ ] Google Sheet received rows with correct columns
- [ ] Data contains unique values (not duplicated brands)
- [ ] Email fields are verified (`valid` or `accept_all` only)
- [ ] Workflow deactivated after testing

## Common Setup Failures

| Symptom | Cause | Fix |
|---------|-------|-----|
| Cannot connect to n8n | Wrong URL or instance down | Verify URL in browser, check `https://` |
| 401 Unauthorized | Invalid API key | Regenerate in n8n Settings → API |
| No credentials found | Not configured in n8n | Add via n8n UI: Settings → Credentials |
| Trigger returns 200 but no execution | Webhook not registered | Cycle activation: deactivate → 3s → activate → 5s |
| Hunter returns "Bad request" for all | Field name mismatch | Check `domain` expression matches upstream output |
| Format Results produces 0 items | All contacts filtered out or field names wrong | Print input field names, check verification filter |
| Sheet node errors | Wrong sheet URL/name or credential issue | Verify sheet URL and credential ID |
| AI node times out | Too many sequential LLM calls | Use Promise.all batches of 12 (see n8n-api-patterns) |

## Related Skills

- **n8n-workflow-building**: Detailed node creation, connection wiring, merge node config
- **n8n-workflow-testing**: Advanced polling, form trigger multipart details, success criteria
- **n8n-workflow-debugging**: Tracing data flow, field mismatch diagnosis, node-by-node inspection
- **n8n-api-patterns**: Code node HTTP, LLM parallelization, 60s timeout workarounds
- **n8n-google-sheets**: Sheet operations, batch writes, cleanup, rate limiting
- **n8n-email-outreach**: Hunter verification, contact ranking, AI cold emails, mail merge

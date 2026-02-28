---
name: n8n-workflow-testing
description: Use when testing n8n workflows end-to-end, triggering form or webhook executions via API, polling execution status, validating node-by-node output, or verifying data reached its destination.
---

# n8n Workflow End-to-End Testing

## Overview

Test n8n workflows programmatically: activate, trigger, poll for completion, and validate every node's output — all without touching the browser UI. A 200 response does NOT mean success.

## When to Use

- After deploying workflow changes and need to verify they work
- Testing form trigger submissions via API
- Monitoring execution progress and diagnosing failures
- Validating that data reached Google Sheets or other destinations

## Core Testing Pattern

### Activate → Baseline → Trigger → Poll → Validate → Deactivate

```python
import requests, time

h = {"X-N8N-API-KEY": API_KEY, "Content-Type": "application/json"}

# 1. Activate
requests.post(f"{BASE}/api/v1/workflows/{WID}/activate", headers=h)
time.sleep(5)

# 2. Record existing execution IDs (baseline)
known = set(str(e["id"]) for e in requests.get(
    f"{BASE}/api/v1/executions?workflowId={WID}&limit=10", headers=h
).json()["data"])

# 3. Trigger (form or webhook)
trigger_workflow(...)

# 4. Poll for NEW execution
new_exec_id = None
for _ in range(40):
    time.sleep(3)
    execs = requests.get(
        f"{BASE}/api/v1/executions?workflowId={WID}&limit=5", headers=h
    ).json()["data"]
    for e in execs:
        if str(e["id"]) not in known:
            new_exec_id = e["id"]
            break
    if new_exec_id:
        break

# 5. Wait for completion
for _ in range(60):
    time.sleep(5)
    ed = requests.get(
        f"{BASE}/api/v1/executions/{new_exec_id}?includeData=true", headers=h
    ).json()
    if ed["status"] in ("success", "error"):
        break

# 6. Validate node-by-node
rd = ed["data"]["resultData"]["runData"]
for name, runs in rd.items():
    for run in runs:
        items = sum(len(o) for o in (run.get("data", {}).get("main") or []) if o)
        err = run.get("error", {}).get("message", "")[:80] if run.get("error") else ""
        print(f"  {name}: {items} items, {run['executionStatus']} {err}")

# 7. Deactivate after testing
requests.post(f"{BASE}/api/v1/workflows/{WID}/deactivate", headers=h)
```

## Triggering Form Triggers

n8n Form Triggers require **multipart/form-data** with an explicit boundary. Python's `files=` shorthand is unreliable.

### Reliable method — manual multipart:
```python
boundary = "----WebKitFormBoundaryXYZ123"
body = "\r\n".join([
    "--" + boundary,
    'Content-Disposition: form-data; name="field-0"',
    "",
    "10",
    "--" + boundary + "--",
    ""
])
requests.post(
    f"{BASE}/form/{WEBHOOK_ID}",
    data=body.encode("utf-8"),
    headers={"Content-Type": f"multipart/form-data; boundary={boundary}"}
)
```

### What does NOT work:
| Method | Result |
|--------|--------|
| `data={"field-0": "10"}` | 500 "Expected multipart/form-data" |
| `json={"data": {"field-0": "10"}}` | 500 "Expected multipart/form-data" |
| `files={"field-0": (None, "10")}` | Sometimes 200 but no execution created |

## Triggering Webhook Triggers

Webhooks are simpler — just POST JSON:

```python
requests.post(f"{BASE}/webhook/{WEBHOOK_PATH}", json={"key": "value"})
```

Production URL: `/webhook/<path>` (workflow must be active)
Test URL: `/webhook-test/<path>` (only during manual test in UI)

## Critical Rule: 200 ≠ Success

Both webhooks and form triggers return 200 as soon as n8n **receives** the request. This does NOT mean:
- The workflow executed successfully
- The workflow even started
- Any nodes produced output

**Always verify** by polling the executions API.

## Validating Execution Results

### Quick node-by-node summary:
```python
rd = ed["data"]["resultData"]["runData"]
for name, runs in rd.items():
    for run in runs:
        total = sum(len(o) for o in (run.get("data", {}).get("main") or []) if o)
        status = run["executionStatus"]
        print(f"  {name}: {total} items ({status})")
```

### Extracting actual data from a node:
```python
node_data = rd["Format Results"][0]["data"]["main"][0]
for item in node_data:
    print(item["json"]["brand_name"], item["json"]["email"])
```

### Checking verification status breakdown:
```python
statuses = {}
for item in rd["Format Results"][0]["data"]["main"][0]:
    s = item["json"].get("verification_status", "unknown")
    statuses[s] = statuses.get(s, 0) + 1
print("Verification breakdown:", statuses)
# e.g., {'valid': 12, 'unknown': 1, 'accept_all': 3}
```

## Success Criteria Checklist

A workflow test passes when:
- [ ] New execution appears within 30 seconds of trigger
- [ ] Execution status is `"success"` (not `"error"` or `"running"` after 5 min)
- [ ] Every node in the pipeline has `executionStatus: "success"`
- [ ] Final destination node (Sheet/API) has item count > 0
- [ ] Data contains **unique** values (not all the same brand repeated)
- [ ] Field names are correct in the output (no `undefined` or empty key fields)

## Playwright MCP Integration

For browser-based validation after API testing:

1. **Verify sheet data** — open the Google Sheet URL and confirm rows were written
2. **Check n8n execution UI** — navigate to the execution and screenshot node outputs
3. **Validate email drafts** — if the workflow generates emails, verify them in Gmail

Use Playwright MCP to automate these visual checks when API validation isn't sufficient.

## Common Testing Mistakes

| Mistake | Fix |
|---------|-----|
| Assuming 200 = success | Always poll executions API for actual status |
| Not recording baseline execution IDs | Record before trigger to identify the NEW execution |
| Polling too fast (< 3s intervals) | Use 3-5 second intervals; full pipeline takes 1-3 minutes |
| Not deactivating after test | Always deactivate to prevent accidental triggers |
| Testing with `webhook-test/` URL on active workflow | Use production `/webhook/` or `/form/` URLs |

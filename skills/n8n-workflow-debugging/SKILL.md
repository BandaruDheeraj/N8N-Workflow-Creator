---
name: n8n-workflow-debugging
description: Use when an n8n workflow fails, nodes produce 0 items, data disappears between nodes, field names don't match, or executions error out. Systematic debugging via REST API.
---

# n8n Workflow Debugging

## Overview

Systematically debug n8n workflows via the REST API. Trace data flow node-by-node, find field mismatches, identify orphaned nodes, and fix the #1 cause of silent data loss: inconsistent field names.

## When to Use

- A workflow execution fails or errors out
- A node receives items but produces 0 output
- Data "disappears" between nodes
- Hunter Search returns "Bad request" for all items
- The workflow runs but nothing gets written to the sheet
- AI node times out after 60 seconds

## 5-Step Debugging Playbook

### Step 1: Fetch the workflow and map connections

```python
wf = requests.get(f"{BASE}/api/v1/workflows/{WID}", headers=h).json()

# Print all nodes
for node in wf["nodes"]:
    print(f"  {node['name']} ({node['type'].split('.')[-1]}) v{node.get('typeVersion', '?')}")

# Print all connections
for src, targets in wf["connections"].items():
    for oidx, outputs in enumerate(targets.get("main", [])):
        for c in outputs:
            print(f"  {src} (out {oidx}) → {c['node']} (in {c['index']})")
```

Look for:
- **Orphaned nodes**: Nodes with no input connections (except triggers)
- **Dangling outputs**: Nodes whose output goes nowhere `[[]]`
- **Wrong input indices**: Data entering a Merge node on the wrong slot

### Step 2: Check recent executions

```python
execs = requests.get(
    f"{BASE}/api/v1/executions?workflowId={WID}&limit=5", headers=h
).json()
for e in execs["data"]:
    print(f"  #{e['id']}: {e['status']} ({e.get('stoppedAt', 'running')})")
```

### Step 3: Trace data flow node-by-node

```python
ed = requests.get(
    f"{BASE}/api/v1/executions/{EXEC_ID}?includeData=true", headers=h
).json()
rd = ed["data"]["resultData"]["runData"]

for name, runs in rd.items():
    for run in runs:
        total = sum(len(o) for o in (run.get("data", {}).get("main") or []) if o)
        err = run.get("error", {}).get("message", "")[:80] if run.get("error") else ""
        print(f"  {name}: {total} items, {run['executionStatus']} {err}")
```

Find where items drop to 0 — that's your problem node.

### Step 4: Inspect the problem node's input vs output

```python
# What went IN
prev_node_data = rd["Previous Node"][0]["data"]["main"][0]
print("Input fields:", list(prev_node_data[0]["json"].keys()))

# What came OUT (if anything)
problem_data = rd["Problem Node"][0]["data"]["main"][0]
print("Output fields:", list(problem_data[0]["json"].keys()))
```

### Step 5: Check the node's code or configuration

```python
node = next(n for n in wf["nodes"] if n["name"] == "Problem Node")
if "jsCode" in node.get("parameters", {}):
    print(node["parameters"]["jsCode"])
else:
    print(json.dumps(node["parameters"], indent=2))
```

## The #1 Bug: Field Name Mismatches

**This is the most common cause of silent data loss.** When one node outputs `domain` but the next expects `brand_domain`, items silently pass through with empty values — no error, no warning.

### Diagnosis:
```python
# Check what fields each node actually outputs
for name, runs in rd.items():
    data = runs[0].get("data", {}).get("main", [[]])[0]
    if data:
        print(f"  {name}: {list(data[0]['json'].keys())}")
```

### Fix: Defensive field access with fallbacks
```javascript
const domain = (item.json.brand_domain || item.json.domain || '').toLowerCase().trim();
const name = item.json.brand_name || item.json.name || 'Unknown';
```

### When renaming fields, update ALL downstream consumers:
```
Parse AI Companies outputs brand_domain
  → Remove Duplicates must check brand_domain
  → Hunter Search expression must use $json.brand_domain
  → Format Results lookup must read item.json.brand_domain
```

## Common Failure Patterns

### Hunter Search returns "Bad request" for all items
**Cause**: Domain parameter expression `={{ $json.domain }}` but input field is `brand_domain`.
**Fix**: Check Hunter node config and match field names:
```python
hunter = next(n for n in wf["nodes"] if n["name"] == "Hunter Search")
print(hunter["parameters"]["domain"])  # shows the expression
```

### Remove Duplicates filters everything to 0
**Cause**: Checking a field that doesn't exist in the input items.
**Fix**: Print actual field names from upstream node and match them.

### AI Code node times out (60s limit on n8n Cloud)
**Cause**: Sequential API calls with delays exceed the hard timeout.
**Fix**: Parallelize with `Promise.all` in batches of 10-12 (see n8n-api-patterns skill).

### splitInBatches immediately fires "done" with 0 items
**Cause**: `typeVersion: 3` bug with stale internal state.
**Fix**: Remove splitInBatches. Wire nodes directly — n8n processes items through downstream nodes automatically.

### Merge node produces Cartesian product (100 items from 10 + 10)
**Cause**: `parameters: {}` or `mode: "combineAll"`.
**Fix**: Set `mode: "append"` explicitly.

### Webhook/form returns 200 but no execution created
**Cause**: After updating trigger nodes, webhook path isn't re-registered.
**Fix**: Cycle activation: deactivate → 3s → activate → 5s.

### `$('NodeName').item` returns undefined
**Cause**: `.item` only works when items flow through a direct chain.
**Fix**: Use `$('NodeName').all()` and build a lookup by key:
```javascript
const lookup = {};
for (const item of $('SomeNode').all()) {
    lookup[item.json.domain] = item.json;
}
const match = lookup[currentDomain];
```

## Quick Reference: Where to Look

| Symptom | Check |
|---------|-------|
| Execution fails | Execution error message + which node failed |
| Node produces 0 items | Input field names vs what the node expects |
| Wrong data in output | Field name mapping between nodes |
| AI node timeout | Code node execution time (>60s = needs parallelization) |
| Sheet not updated | Final node item count + sheet API errors |
| Webhook 404 | Activation cycling needed |
| Duplicate data | Dedup node field reference + existing leads query |

---
name: n8n-api-patterns
description: Use when working with n8n Code node limitations, httpRequest in Code nodes, OpenRouter LLM calls with parallelization, 60-second timeout workarounds, or credential management via REST API.
---

# n8n API & Code Node Patterns

## Overview

Patterns for working within n8n's constraints: Code node HTTP capabilities, the 60-second Cloud timeout, LLM call parallelization, credential management, and the quirks that aren't in the docs.

## When to Use

- Making HTTP requests inside n8n Code nodes
- Calling OpenRouter/LLM APIs from workflows
- Hitting the 60-second Code node timeout
- Managing credentials and authentication
- Generating data externally and pushing through n8n

## Code Node HTTP Capabilities

| Method | Status | Notes |
|--------|--------|-------|
| `this.helpers.httpRequest()` | ✅ Works | Basic HTTP calls |
| `this.helpers.httpRequestWithAuthentication()` | ❌ Broken | Despite docs, does NOT work in Code nodes |
| `fetch()` | ❌ Unavailable | Not in n8n's sandbox |
| `globalThis.fetch` | ❌ Unavailable | Not in n8n's sandbox |
| `$http` | ❌ Unavailable | Not defined |
| `require()` | ✅ Available | Can import built-in Node.js modules |

### Making API calls in Code nodes:
```javascript
const response = await this.helpers.httpRequest({
    method: 'POST',
    url: 'https://openrouter.ai/api/v1/chat/completions',
    headers: {
        'Authorization': 'Bearer ' + apiKey,
        'Content-Type': 'application/json'
    },
    body: JSON.stringify({
        model: 'anthropic/claude-sonnet-4.6',
        messages: [{ role: 'system', content: '...' }, { role: 'user', content: '...' }],
        max_tokens: 800,
        temperature: 0.7
    })
});
const data = typeof response === 'string' ? JSON.parse(response) : response;
```

### For authenticated Google API calls:
Use an **HTTP Request node** (not Code node) with `authentication: "predefinedCredentialType"` and `nodeCredentialType: "googleSheetsOAuth2Api"`.

## The 60-Second Timeout Problem

n8n Cloud enforces a **hard 60-second limit** on Code node execution. Sequential LLM calls will exceed this for 5+ items.

### Wrong — sequential with delays:
```javascript
// 24 items × 5s each = 120s → TIMEOUT
for (const item of items) {
    const resp = await this.helpers.httpRequest({...});
    await sleep(3000);
}
```

### Right — parallel batches with Promise.all:
```javascript
const BATCH_SIZE = 12;
const allResults = [];

async function processItem(row, self) {
    const resp = await self.helpers.httpRequest({
        method: 'POST',
        url: 'https://openrouter.ai/api/v1/chat/completions',
        headers: { 'Authorization': 'Bearer ' + API_KEY, 'Content-Type': 'application/json' },
        body: JSON.stringify({
            model: 'anthropic/claude-sonnet-4.6',
            messages: [{ role: 'user', content: `Write a pitch for ${row.brand_name}` }],
            max_tokens: 800
        })
    });
    const data = typeof resp === 'string' ? JSON.parse(resp) : resp;
    return { json: { ...row, pitch: data.choices[0].message.content } };
}

for (let i = 0; i < items.length; i += BATCH_SIZE) {
    const batch = items.slice(i, i + BATCH_SIZE);
    const results = await Promise.all(
        batch.map(item => processItem(item.json, this))
    );
    allResults.push(...results);
    if (i + BATCH_SIZE < items.length) await sleep(1000);
}
return allResults;
```

**Critical**: Pass `this` as `self` parameter — arrow functions inside `Promise.all` lose the `this` binding, so `self.helpers.httpRequest()` is needed.

### When parallelization isn't enough (50+ items):
Generate data externally in Python, then push through an n8n webhook:
```python
# Python generates AI content
for brand in brands:
    result = openrouter_call(brand)
    results.append(result)

# Push through n8n webhook in batches of 30
for i in range(0, len(results), 30):
    batch = results[i:i+30]
    requests.post(f"{BASE}/webhook/backfill", json=batch)
    time.sleep(3)
```

## Rate Limiting Strategy

| Model Tier | Rate Limit | Batch Size | Notes |
|------------|-----------|------------|-------|
| Free OpenRouter | ~10 req/min shared | 1-2 | Unreliable, frequent timeouts |
| Paid OpenRouter | Practically unlimited | 12 concurrent | Claude Sonnet 4.6, DeepSeek v3 |
| Google Sheets API | ~60 req/min | 30 rows per batch | 3s delay between batches |
| Hunter.io | Plan-dependent | n8n handles automatically | Check plan limits |

## Credential Management

### List available credentials:
```python
creds = requests.get(f"{BASE}/api/v1/credentials", headers=h).json()
for c in creds["data"]:
    print(f"  {c['id']}: {c['name']} ({c['type']})")
```

### Reference credentials in nodes:
```python
node["credentials"] = {
    "googleSheetsOAuth2Api": {"id": "<your-credential-id>", "name": "Google Sheets account"},
    "hunterApi": {"id": "<your-credential-id>", "name": "Hunter account"},
    "openRouterApi": {"id": "abc123", "name": "OpenRouter account"}
}
```

## Temporary Workflow Pattern

When you need a one-off operation (read sheet data, clear a sheet), create a temp workflow:

```python
# 1. Save original state of utility workflow
original_nodes = json.loads(json.dumps(wf["nodes"]))
original_connections = json.loads(json.dumps(wf["connections"]))

# 2. Replace with one-off operation
# ... modify nodes ...

# 3. Deploy, trigger, get result
# 4. ALWAYS restore original state
wf["nodes"] = original_nodes
wf["connections"] = original_connections
```

**Critical**: Keep the original Webhook node object intact (same `id`, `webhookId`, `name`). Replacing it causes webhook path de-registration → 404 errors.

## PowerShell/Python Escaping

When running Python inline from PowerShell, complex strings with nested quotes break. **Always write to a .py file first**:

```python
# Write the script to a file
with open('fix_node.py', 'w') as f:
    f.write(script_content)
# Then execute: python fix_node.py
```

Never try to inline n8n Code node JavaScript (with template literals, regex, JSON) into a PowerShell `-c` command.

## Quick Reference

| Need | Pattern |
|------|---------|
| HTTP call in Code node | `this.helpers.httpRequest()` |
| Authenticated Google call | HTTP Request node (not Code node) |
| LLM calls for 5+ items | `Promise.all` in batches of 12 |
| LLM calls for 50+ items | Python external + webhook push |
| One-off sheet operation | Temp workflow: modify → execute → restore |
| After any workflow change | Deactivate → 3s → activate → 5s |

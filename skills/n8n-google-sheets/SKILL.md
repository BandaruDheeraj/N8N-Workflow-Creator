---
name: n8n-google-sheets
description: Use when integrating n8n workflows with Google Sheets — appending rows, updating existing data, creating tabs, batch operations, cleaning up garbage data, or fixing column misalignment.
---

# n8n Google Sheets Integration

## Overview

Patterns for reliable Google Sheets operations in n8n: appending rows, batch updates, creating tabs, cleaning up accumulated garbage data, and avoiding the pitfalls of rate limits and column misalignment.

## When to Use

- Writing workflow output to Google Sheets
- Updating existing rows matched by a key column
- Creating new sheet tabs programmatically
- Cleaning up sheets with bad/test/duplicate data
- Batch operations on 30+ rows with rate limiting

## Sheet Reference Modes

```python
# Document ID — use mode "url" with full URL
"documentId": {"mode": "url", "value": "https://docs.google.com/spreadsheets/d/DOC_ID"}

# Sheet name — use mode "name" with tab name
"sheetName": {"mode": "name", "value": "High Priority Email"}

# Sheet by GID — use mode "id" with just the number (NOT "gid=12345")
"sheetName": {"mode": "id", "value": "1863718260"}
```

## Append Operation

Adds new rows. Uses `autoMapInputData` — any JSON field name matching a column header becomes a cell value.

```python
{
    "parameters": {
        "operation": "append",
        "documentId": {"mode": "url", "value": SHEET_URL},
        "sheetName": {"mode": "name", "value": "My Sheet"},
        "columns": {
            "mappingMode": "autoMapInputData",
            "value": {}
        },
        "options": {}
    },
    "type": "n8n-nodes-base.googleSheets",
    "typeVersion": 4.5,
    "credentials": {
        "googleSheetsOAuth2Api": {"id": CRED_ID, "name": "Google Sheets account"}
    }
}
```

## Update Operation

Modifies existing rows matched by a key column. Requires `matchingColumns` and a `schema` with `canBeUsedToMatch: True`.

```python
{
    "parameters": {
        "operation": "update",
        "documentId": {"mode": "url", "value": SHEET_URL},
        "sheetName": {"mode": "id", "value": GID},
        "columns": {
            "mappingMode": "autoMapInputData",
            "value": {},
            "matchingColumns": ["email"],
            "schema": [
                {"id": "email", "displayName": "email", "required": False,
                 "defaultMatch": True, "display": True, "type": "string",
                 "canBeUsedToMatch": True},
            ]
        },
        "options": {}
    }
}
```

**Common error**: `"The 'Column to Match On' parameter is required"` means `matchingColumns` is missing or the schema doesn't have `canBeUsedToMatch: True`.

## Creating New Sheet Tabs

The append/update operations do NOT auto-create tabs. Use an HTTP Request node:

```python
http_request_node = {
    "parameters": {
        "method": "POST",
        "url": f"=https://sheets.googleapis.com/v4/spreadsheets/{DOC_ID}:batchUpdate",
        "authentication": "predefinedCredentialType",
        "nodeCredentialType": "googleSheetsOAuth2Api",
        "sendBody": True,
        "specifyBody": "json",
        "jsonBody": json.dumps({
            "requests": [{"addSheet": {"properties": {"title": "New Tab Name"}}}]
        })
    },
    "credentials": {
        "googleSheetsOAuth2Api": {"id": CRED_ID, "name": "Google Sheets account"}
    }
}
```

**Note**: `this.helpers.httpRequestWithAuthentication()` does NOT work in Code nodes for Google API calls. You must use an HTTP Request node.

## Batch Operations via Webhook

For large operations (30+ rows), push data through an n8n webhook in batches:

```python
BATCH_SIZE = 30
for i in range(0, len(data), BATCH_SIZE):
    batch = data[i:i + BATCH_SIZE]
    requests.post(f"{BASE}/webhook/{WEBHOOK_PATH}", json=batch, timeout=60)
    time.sleep(3)  # Google Sheets API rate limit
```

The receiving webhook workflow:
```
Webhook → Parse Data (Code node) → Append/Update Sheet
```

Parse Data code:
```javascript
const body = $input.first().json.body;
if (Array.isArray(body)) {
    return body.map(item => ({ json: item }));
}
return [{ json: body }];
```

**Always verify** execution status after each batch — 200 from webhook ≠ successful sheet write.

## Sheet Cleanup Pattern

Sheets accumulate garbage from failed runs, test data, and partial executions. Clean periodically.

### Step 1: Audit
```python
import csv, io, urllib.request

url = f'https://docs.google.com/spreadsheets/d/{DOC_ID}/gviz/tq?tqx=out:csv&sheet={SHEET_NAME}'
data = urllib.request.urlopen(url).read().decode('utf-8')
rows = list(csv.DictReader(io.StringIO(data)))

no_email = [r for r in rows if not r.get('email', '').strip()]
brands = [r.get('brand_name', '') for r in rows]
dupes = [b for b in set(brands) if brands.count(b) > 1]
print(f"Total: {len(rows)}, No email: {len(no_email)}, Dupes: {len(dupes)}")
```

### Step 2: Filter in Python
```python
seen = set()
clean = []
for row in rows:
    if not row.get('email', '').strip():
        continue  # skip no-email rows
    brand = row.get('brand_name', '').strip()
    if brand in seen:
        continue  # skip duplicate brands
    seen.add(brand)
    # Fix known data issues
    if brand == '1' and '100percent' in row.get('brand_domain', ''):
        row['brand_name'] = '100%'
    clean.append(row)
```

### Step 3: Clear sheet (keep headers) via temp workflow
Create a temporary workflow with a Google Sheets `clear` operation:
```python
clear_node = {
    "parameters": {
        "operation": "clear",
        "documentId": {"mode": "url", "value": SHEET_URL},
        "sheetName": {"mode": "name", "value": SHEET_NAME},
        "clear": "exceptHeaders"
    }
}
```

### Step 4: Rewrite clean data via webhook batches

### Prevention: Validate BEFORE writing
Add guards in pre-sheet Code nodes:
```javascript
const valid = results.filter(r =>
    r.json.email && r.json.email.trim() &&
    r.json.brand_name && r.json.brand_name.trim()
);
if (valid.length === 0) {
    console.log('No valid rows — skipping sheet write');
    return [];
}
return valid;
```

## Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Shifted columns | Phantom column (e.g., `row_number`) inserted | Delete phantom column, verify alignment |
| Duplicate rows | Multiple runs finding same brands | Add dedup logic before sheet write |
| Truncated values | Sheets auto-format (e.g., "100%" → "1") | Fix programmatically in cleanup |
| Rate limit errors | Too many batch requests | 30 rows per batch, 3s delay between |
| `matchingColumns` error | Missing schema with `canBeUsedToMatch` | Add full schema array to update node |
| Tab doesn't exist | Append can't auto-create tabs | Use HTTP Request node with batchUpdate API |

---
name: n8n-email-outreach
description: Use when building email outreach systems with n8n — Hunter.io email verification, contact ranking and scoring, AI-generated personalized cold emails, mail merge setup with YAMM or Mailmeteor.
---

# n8n Email Outreach Systems

## Overview

Build automated email outreach pipelines in n8n: discover contacts via Hunter.io, filter to verified emails only, rank contacts by relevance, generate personalized AI cold emails, and output YAMM/Mailmeteor-ready Google Sheets.

## When to Use

- Building lead generation workflows with Hunter.io
- Filtering contacts to only verified/deliverable emails
- Ranking multiple contacts per company for outreach priority
- Generating personalized cold emails via LLM
- Setting up Google Sheets for mail merge tools

## Hunter.io Email Verification

### The free approach (no extra credits)

Every contact from Hunter's Domain Search already includes verification data:
```json
{
    "value": "jane@company.com",
    "verification": { "date": "2026-02-18", "status": "valid" }
}
```

### Verification status values

| Status | Meaning | Action |
|--------|---------|--------|
| `valid` | Email is deliverable | ✅ Include |
| `accept_all` | Server accepts all (can't verify individually) | ✅ Include |
| `invalid` | Email bounces | ❌ Filter out |
| `disposable` | Temporary/throwaway | ❌ Filter out |
| `webmail` | Personal email (gmail, etc.) | ⚠️ Exclude for B2B |
| `unknown` | Could not verify | ❌ Filter out |

### Extract in Format Results node:
```javascript
for (const contact of data.emails) {
    results.push({
        json: {
            brand_name: brand.name,
            email: contact.value || '',
            position: contact.position || '',
            confidence: contact.confidence || '',
            verification_status: (contact.verification && contact.verification.status) || 'unknown',
            // ... other fields
        }
    });
}
```

### Filter in downstream node:
```javascript
const VALID_STATUSES = ['valid', 'accept_all'];
const verified = items.filter(item => {
    const status = (item.json.verification_status || 'unknown').toLowerCase();
    return VALID_STATUSES.includes(status);
});
console.log('Verified: ' + verified.length + ' of ' + items.length);
```

**Filter BEFORE scoring/ranking**, not after. Brands with zero verified contacts are automatically excluded.

### Don't waste credits on re-verification
The separate Hunter Email Verification API (1 credit/email) is unnecessary for fresh Domain Search results. Only use it for contacts that are months old.

## Contact Ranking Algorithm

When Hunter returns multiple contacts per company, rank them to pick the best recipient(s):

```javascript
const TITLE_HIGH = ['partnership', 'sponsorship', 'brand ambassador',
    'brand marketing', 'cmo', 'vp marketing', 'head of marketing',
    'head of partnerships', 'director of marketing'];           // +15 pts
const TITLE_MED = ['marketing', 'brand', 'communications',
    'social media', 'content', 'director', 'vp'];              // +8 pts
const TITLE_LOW = ['manager', 'coordinator', 'specialist'];    // +3 pts
const SENIOR = ['chief', 'ceo', 'coo', 'cmo', 'founder', 'president']; // +10 pts
const MID_SENIOR = ['vp', 'vice president', 'director'];       // +7 pts
const DEPT_SCORES = { marketing: 5, executive: 4, communications: 3, sales: 2 };

function scoreContact(c) {
    let score = 0;
    const pos = (c.position || '').toLowerCase();
    const dept = (c.department || '').toLowerCase();

    if (TITLE_HIGH.some(k => pos.includes(k))) score += 15;
    else if (TITLE_MED.some(k => pos.includes(k))) score += 8;
    else if (TITLE_LOW.some(k => pos.includes(k))) score += 3;

    if (SENIOR.some(s => pos.includes(s))) score += 10;
    else if (MID_SENIOR.some(s => pos.includes(s))) score += 7;

    for (const [d, pts] of Object.entries(DEPT_SCORES)) {
        if (dept.includes(d)) { score += pts; break; }
    }

    score += Math.floor((parseInt(c.confidence || '0') || 0) / 25); // 0-4 pts
    if (c.linkedin && c.linkedin.trim()) score += 2;
    if (pos.trim()) score += 1;
    return score;
}
```

### Select top contacts per brand:
```javascript
// Group by brand → sort by score → pick top 1 (To) + next 2 (CC)
const primary = scored[0].contact;
const ccContacts = scored.slice(1, 3).map(s => s.contact);

// Build greeting addressing all recipients
let greeting = 'Hey ' + primary.first_name;
if (ccFirstNames.length === 1) greeting += ' and ' + ccFirstNames[0];
else if (ccFirstNames.length === 2) greeting += ', ' + ccFirstNames[0] + ', and ' + ccFirstNames[1];
```

## AI Email Generation

### Pipeline order (critical):
```
Format Results → Select Priority Contacts → AI Generate Emails → Add to Sheet
```
The AI MUST know who the recipients are before writing the email. Select contacts FIRST, then generate.

### Prompt design:
```javascript
const systemPrompt = `You are writing cold sponsorship outreach emails.
Return JSON: {"pitch": "...", "subject": "..."}
Do NOT include any sign-off or signature.
Keep it concise — 3-4 sentences for the pitch.`;

const userPrompt = `Brand: ${row.brand_name} (${row.category})
Recipients: ${row.greeting}
Primary contact: ${row.name}, ${row.position}`;
```

### Parallelization required for 5+ brands:
Use `Promise.all` in batches of 12 to stay within the 60s Code node timeout. See the n8n-api-patterns skill.

### Email structure:
```
Subject: {Brand} × {Team Name} | Seeking Sponsors

{Greeting with all contact names},

{1-2 sentence intro}

{AI-generated brand-specific pitch}

{Value proposition list}

{Sponsorship ask — monetary + gear/product}

(NO signature — user adds in email client)
```

## Mail Merge Sheet Format

Output a Google Sheet that works directly with YAMM or Mailmeteor:

| Column | Purpose | Example |
|--------|---------|---------|
| `email` | Primary recipient | jane@company.com |
| `cc` | CC recipients (comma-separated) | bob@company.com, alice@company.com |
| `subject` | Email subject line | Hoka × Speed Project Team \| Seeking Sponsors |
| `body` | Full email body (no signature) | Hey Jane and Bob, ... |
| `name` | Primary contact name | Jane Smith |
| `position` | Primary contact title | VP Marketing |
| `brand_name` | For reference/filtering | Hoka |

### Using with YAMM/Mailmeteor:
1. Create a Gmail draft with merge tags: `{{subject}}` in subject, `{{body}}` in body
2. Open the sheet → Extensions → YAMM/Mailmeteor → Start mail merge
3. Select `email` as recipient column — `cc` is auto-detected
4. Send test batch of 3-5 first, then send all

### Don't put signatures in the sheet
Keep the `body` column clean — the user adds their own signature block in the Gmail draft template. If the AI generates a sign-off, add `"Do NOT include any sign-off or signature"` to the system prompt.

## AI Brand Discovery + Dedup

### Over-request to account for duplicates:
The AI often generates brands already in your sheet despite exclusion lists. Request 2x what you need:
- Need 11 new brands → request 20-25 from the AI
- Dedup filters to truly new ones
- Hunter may not find contacts for some → further reduction

### Dynamic exclusion list:
```javascript
const existing = $('Read Existing Leads').all();
const names = [...new Set(existing.map(i => i.json.brand_name))];
const prompt = `Generate ${count} companies. Do NOT include: ${names.join(', ')}`;
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| AI generates email before knowing recipients | Reorder: Select Contacts → AI Emails |
| Not filtering invalid Hunter emails | Add verification_status filter |
| Including AI signature in sheet body | Add "no sign-off" to system prompt |
| Exclusion list only has seed brands | Feed ALL existing leads into AI prompt |
| Sending to unverified emails | Only allow `valid` + `accept_all` status |

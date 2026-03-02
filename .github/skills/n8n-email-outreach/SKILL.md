---
name: n8n-email-outreach
description: Use when building email outreach systems with n8n — Hunter.io email verification, contact ranking and scoring, AI-generated personalized cold emails with humanization to avoid AI-sounding patterns, mail merge setup with YAMM or Mailmeteor.
---

# n8n Email Outreach Systems

## Overview

Build automated email outreach pipelines in n8n: discover contacts via Hunter.io, filter to verified emails only, rank contacts by relevance, generate personalized AI cold emails that sound human (not AI-generated), and output YAMM/Mailmeteor-ready Google Sheets.

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

### Prompt design (with humanization rules):
```javascript
const systemPrompt = `You are writing cold sponsorship outreach emails.
Return JSON: {"pitch": "...", "subject": "..."}
Do NOT include any sign-off or signature.
Keep it concise — 3-4 sentences for the pitch.

CRITICAL — Write like a human, not an AI. Follow these rules:
- No filler phrases: "In order to", "Due to the fact that", "It is important to note"
- No significance inflation: "pivotal", "transformative", "groundbreaking", "game-changing"
- No AI vocabulary: "delve", "landscape", "tapestry", "foster", "leverage", "underscore", "showcase"
- No promotional fluff: "vibrant", "nestled", "renowned", "stunning", "breathtaking"
- No sycophantic tone: "Great question!", "Absolutely!", "I'd love to"
- No rule-of-three lists: "innovation, inspiration, and insights"
- No negative parallelisms: "It's not just X, it's Y"
- No em dash overuse — use commas or periods instead
- No excessive hedging: "could potentially", "might possibly"
- No generic conclusions: "The future looks bright", "Exciting times ahead"
- Use "is/are/has" instead of "serves as/stands as/boasts/features"
- Vary sentence length. Short ones. Then a longer one that takes its time.
- Be specific: real numbers, concrete details, not vague claims
- Sound like a person who typed this in 2 minutes, not a bot that generated it`;

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

## Humanizing AI-Generated Emails

Cold emails written by AI get ignored. Recipients — especially marketing professionals — can spot AI-generated text instantly. A humanization pass dramatically improves open and reply rates.

Based on [Wikipedia's "Signs of AI writing"](https://en.wikipedia.org/wiki/Wikipedia:Signs_of_AI_writing) guide and the [humanizer](https://github.com/blader/humanizer) project.

### The 12 Patterns That Kill Cold Email Engagement

These are the AI tells most damaging in outreach emails, ordered by how quickly they get you deleted:

| # | Pattern | AI Version | Human Version |
|---|---------|-----------|---------------|
| 1 | **Sycophantic opener** | "I'd love to explore a partnership!" | "Quick question about sponsorships." |
| 2 | **Significance inflation** | "transformative partnership opportunity" | "sponsorship" |
| 3 | **Promotional fluff** | "your vibrant, industry-leading brand" | "your brand" |
| 4 | **AI vocabulary** | "delve into synergies", "leverage your platform" | "talk about working together" |
| 5 | **Filler phrases** | "In order to explore potential alignment" | "To see if this fits" |
| 6 | **Rule of three** | "visibility, engagement, and growth" | "visibility with our audience" |
| 7 | **Em dash overuse** | "your brand — known for innovation — could..." | "your brand could..." |
| 8 | **Negative parallelism** | "It's not just sponsorship, it's a movement" | State the value directly |
| 9 | **Generic conclusion** | "Exciting times ahead for both our teams!" | "Let me know if you're open to a call." |
| 10 | **Excessive hedging** | "We were wondering if you might potentially..." | "Would you be open to..." |
| 11 | **Copula avoidance** | "Your brand serves as a beacon of..." | "Your brand is..." |
| 12 | **Synonym cycling** | "partnership... collaboration... alliance..." | "partnership" (just repeat it) |

### Two-Pass Humanization in n8n Code Node

After AI generates the initial email, run a second LLM pass to audit and rewrite. This catches patterns the generation prompt missed.

#### Pipeline with humanization:
```
Select Priority Contacts → AI Generate Emails → Humanize Emails → Add to Sheet
```

#### Humanize Emails Code node:
```javascript
const BATCH_SIZE = 12;
const API_KEY = '<OPENROUTER_KEY>';
const allResults = [];

async function humanizeEmail(row, self) {
    const resp = await self.helpers.httpRequest({
        method: 'POST',
        url: 'https://openrouter.ai/api/v1/chat/completions',
        headers: {
            'Authorization': 'Bearer ' + API_KEY,
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            model: 'anthropic/claude-sonnet-4.6',
            messages: [
                {
                    role: 'system',
                    content: `You are an editor who removes AI-sounding patterns from cold emails.
Return JSON: {"pitch": "...", "subject": "..."}

Rewrite the email to fix these problems if present:
- Sycophantic tone ("I'd love to", "excited to", "thrilled")
- Significance inflation ("transformative", "pivotal", "groundbreaking")
- Promotional fluff ("vibrant", "renowned", "industry-leading")
- AI vocabulary ("delve", "leverage", "foster", "landscape", "synergy")
- Filler phrases ("In order to", "It is worth noting")
- Rule of three lists
- Em dash overuse
- "serves as" / "stands as" instead of "is"
- Excessive hedging ("might potentially")
- Generic positive conclusions

Make it sound like a real person dashed off a quick email.
Vary sentence length. Be specific. Keep the same meaning.
Do NOT add a sign-off or signature.`
                },
                {
                    role: 'user',
                    content: `Subject: ${row.subject}\n\nBody: ${row.body}`
                }
            ],
            max_tokens: 600,
            temperature: 0.8
        })
    });
    const data = typeof resp === 'string' ? JSON.parse(resp) : resp;
    const text = data.choices[0].message.content;
    const match = text.match(/\{[\s\S]*\}/);
    const result = JSON.parse(match[0]);
    return { json: { ...row, subject: result.subject, body: result.pitch } };
}

const items = $input.all();
for (let i = 0; i < items.length; i += BATCH_SIZE) {
    const batch = items.slice(i, i + BATCH_SIZE);
    const results = await Promise.all(
        batch.map(item => humanizeEmail(item.json, this))
    );
    allResults.push(...results);
    if (i + BATCH_SIZE < items.length) await sleep(1000);
}
return allResults;
```

### Before/After Examples

**Before (AI-generated):**
> Subject: Hoka × Speed Project | Transformative Partnership Opportunity
>
> Hey Jane and Bob,
>
> I'd love to explore an exciting partnership between Hoka and the Speed Project team. Your brand serves as a beacon of innovation in the running space, and we believe there's a tremendous opportunity to leverage our combined platforms to foster deeper community engagement, drive brand visibility, and create meaningful impact.
>
> We'd be thrilled to delve into how this collaboration could align with your marketing objectives. Exciting times ahead!

**After (humanized):**
> Subject: Hoka × Speed Project | Sponsorship
>
> Hey Jane and Bob,
>
> We run ultramarathon relays across the desert — 340 miles, no sleep, small teams. Our runners already wear Hoka. It'd make sense to make that official.
>
> We get about 2M social impressions per race and 15K email subscribers. Happy to send our media kit if you're open to a quick call.

### Prompt-Only Approach (No Second Pass)

If you want to skip the extra Code node and LLM call, the humanization rules in the generation prompt (see "Prompt design" above) handle most patterns. The two-pass approach catches ~30% more AI tells but doubles LLM cost.

### Quick Humanization Checklist

Run through this mentally (or in code) before sending:

- [ ] No "I'd love to" or "excited to" or "thrilled to"
- [ ] No "transformative/pivotal/groundbreaking/game-changing"
- [ ] No "delve/leverage/foster/synergy/landscape"
- [ ] No three-item lists ending in "and [abstract noun]"
- [ ] Subject line is short and specific, not salesy
- [ ] Email reads like it took 2 minutes to write, not 2 hours
- [ ] Sentences vary in length (not all the same rhythm)
- [ ] Specific numbers or details, not vague claims
- [ ] No sign-off or signature (user adds their own)

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
| Emails sound AI-generated (low reply rate) | Add humanization rules to system prompt or use two-pass rewrite |
| Humanize node times out | Use Promise.all batches of 12 with `self` binding (see n8n-api-patterns) |
| Over-humanizing removes brand specifics | Check that rewrite still mentions the brand name and concrete details |

## References

- [Wikipedia: Signs of AI writing](https://en.wikipedia.org/wiki/Wikipedia:Signs_of_AI_writing) — Primary source for humanization patterns
- [blader/humanizer](https://github.com/blader/humanizer) — 24-pattern detection skill this section is based on
- [WikiProject AI Cleanup](https://en.wikipedia.org/wiki/Wikipedia:WikiProject_AI_Cleanup) — Maintaining organization

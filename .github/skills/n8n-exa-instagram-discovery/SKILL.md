---
name: n8n-exa-instagram-discovery
description: Use when finding Instagram profiles of athletes, coaches, or niche communities using Exa Search API. Discovers verified handles from Instagram URLs instead of guessing/hallucinating. Covers n8n Code node integration, handle extraction, category balancing, and brand filtering.
---

# Exa Search API — Instagram Profile Discovery for n8n

## Overview

Discover real Instagram profiles for niche communities (athletes, coaches, creators) using Exa's neural search API. Instead of the broken approach of "name → guess Instagram handle" (which hallucinates), this **flips the search**: query Instagram for content in your niche and extract verified handles directly from URLs.

**Key insight**: Searching `includeDomains: ["instagram.com"]` returns Instagram post/profile URLs. The handle IS in the URL — no guessing needed.

## When to Use

- Finding Instagram profiles of athletes, coaches, or niche communities
- Building outreach lists where you need **verified** (not hallucinated) handles
- Replacing Perplexity/Claude/GPT "find their Instagram" prompts that hallucinate
- Discovering potential clients/users in specific fitness, sports, or hobby niches
- Any n8n workflow that needs real Instagram handles at scale

## Why NOT to Use Name → Handle Lookup

We tested every approach. Here's what fails and why:

| Approach | Problem |
|----------|---------|
| Perplexity/Sonar "find X's Instagram" | Hallucinates handles for obscure people (~40% fake) |
| Exa `includeDomains: ["instagram.com"]` + person name | Returns posts BY other people that mention the topic, not the target person |
| Exa `includeText` with person's last name | Matches wrong people with same name fragments |
| Exa open web search + extract IG links from content | Obscure people don't have web pages linking to their IG |
| Exa `findSimilar` from a known profile | Matches by username similarity, not content |

**What works**: Search Instagram for CONTENT in your niche → extract handles from URLs.

## Exa Search API vs Exa Websets

| | Exa Search API ✅ | Exa Websets ❌ |
|---|---|---|
| **What** | Live neural web search | Entity list builder |
| **Pricing** | $0.005/search (1-25 results) | 10 credits/matched result |
| **Best for** | Finding URLs on specific domains | B2B lead enrichment |
| **For IG discovery** | Perfect — filter to instagram.com | Overkill and expensive |

**Always use the Search API**, not Websets, for Instagram discovery.

## Core Pattern: n8n Code Node with Exa

### The Discovery Code Node (JavaScript)

```javascript
const EXA_KEY = "your-exa-api-key";
const self = this;

// Define search queries — one per niche/category you want
const queries = [
  {q: "HYROX athlete training race competition results", cat: "HYROX Athlete"},
  {q: "triathlon athlete ironman training swim bike run race day", cat: "Triathlete"},
  {q: "marathon runner training race personal best half marathon", cat: "Runner"},
  {q: "trail running ultra marathon athlete mountain race finish", cat: "Trail Runner"},
  {q: "running coach training plan 5K 10K speed work interval", cat: "Running Coach"},
  {q: "fitness coach online personal training hybrid athlete", cat: "Fitness Coach"},
];

const results = [];
const seenHandles = new Set();

// Skip known brand/org accounts — you want individuals
const skipHandles = new Set([
  "p", "reel", "reels", "explore", "stories", "accounts",
  "popular", "tags", "about", "directory",
  "hyrox", "hyroxworld", "ironmantri", "crossfit", "crossfitgames",
  "nike", "nikerunning", "adidas", "redbull", "garmin", "strava",
  "runnersworldmag"
]);

for (const {q, cat} of queries) {
  try {
    const resp = await self.helpers.httpRequest({
      method: "POST",
      url: "https://api.exa.ai/search",
      headers: {
        "x-api-key": EXA_KEY,
        "Content-Type": "application/json"
      },
      body: JSON.stringify({
        query: q,
        type: "neural",
        numResults: 25,        // Max 25 for $0.005/search
        includeDomains: ["instagram.com"]
      })
    });

    const data = typeof resp === "string" ? JSON.parse(resp) : resp;
    const hits = data.results || [];

    for (const r of hits) {
      const url = r.url || "";
      const title = r.title || "";

      // Extract handle from URL
      let handle = null;

      // Pattern 1: instagram.com/{handle}/p/... or /reel/...
      const m1 = url.match(/instagram\.com\/([a-zA-Z0-9_.]{2,30})\/(p|reel|reels)\//);
      if (m1) handle = m1[1];

      // Pattern 2: profile page instagram.com/{handle}/
      if (!handle) {
        const m2 = url.match(/instagram\.com\/([a-zA-Z0-9_.]{2,30})\/?$/);
        if (m2) handle = m2[1];
      }

      if (!handle || skipHandles.has(handle.toLowerCase()) || seenHandles.has(handle.toLowerCase())) continue;
      seenHandles.add(handle.toLowerCase());

      // Extract display name from title
      let name = "";
      const nameMatch = title.match(/^([^|(@\n]+?)\s*(?:on Instagram|\(@|\|)/);
      if (nameMatch) {
        name = nameMatch[1].trim().replace(/[^\w\s.'-]/g, "").trim();
      }
      if (!name || name.length < 3) name = handle;

      results.push({json: {
        name: name,
        instagram: handle,
        email: "",
        category: cat
      }});
    }
  } catch (e) {
    console.log("Exa error: " + String(e.message || e).substring(0, 100));
  }
}

return results.length > 0 ? results : [{json: {error: "No profiles found"}}];
```

### Handle Extraction Rules

Instagram URLs come in these patterns:

| URL Pattern | Handle Location |
|-------------|-----------------|
| `instagram.com/{handle}/p/ABC123/` | ✅ First path segment |
| `instagram.com/{handle}/reel/ABC123/` | ✅ First path segment |
| `instagram.com/{handle}/` | ✅ First path segment (profile page) |
| `instagram.com/p/ABC123/` | ❌ No handle — skip |
| `instagram.com/reel/ABC123/` | ❌ No handle — skip |
| `instagram.com/explore/tags/...` | ❌ Not a profile — skip |

**Critical**: Many results will be `instagram.com/p/...` with no handle. Expect ~50-60% of results to have extractable handles. Plan query count accordingly.

### Name Extraction from Title

Instagram page titles follow these patterns:
```
"Jake Gibbs on Instagram: "HYROX ANAHEIM recap..."   → name = "Jake Gibbs"
"Sean Noble (@sean_noble7) • Instagram photos"        → name = "Sean Noble"
"Kirsty Hendey - Hybrid Coach on Instagram: "..."     → name = "Kirsty Hendey - Hybrid Coach"
"Instagram"                                            → name = (use handle as fallback)
```

Regex: `/^([^|(@\n]+?)\s*(?:on Instagram|\(@|\|)/`

## Category Balancing

If you run 12+ queries, the first categories dominate (HYROX fills 50 slots before triathlon queries process). Use round-robin allocation:

```javascript
// Group by category
const byCategory = {};
for (const item of items) {
    const cat = item.json.category || "Other";
    if (!byCategory[cat]) byCategory[cat] = [];
    byCategory[cat].push(item);
}

// Round-robin: max N per category first, then fill remaining
const maxPerCat = 8;  // For 7 categories × 8 = 56 max, capped at 50
const result = [];
for (const cat of Object.keys(byCategory)) {
    result.push(...byCategory[cat].slice(0, maxPerCat));
}
// Fill remaining slots from overflow
if (result.length < 50) {
    for (const cat of Object.keys(byCategory)) {
        for (const item of byCategory[cat].slice(maxPerCat)) {
            if (result.length >= 50) break;
            result.push(item);
        }
    }
}
```

## Brand/Org Filtering

Exa returns brand accounts mixed with individuals. Filter these:

**Always skip** (add to `skipHandles`):
- Official sport accounts: `hyroxworld`, `hyroxamerica`, `ironmantri`, `crossfitgames`
- Shoe/gear brands: `nike`, `nikerunning`, `adidas`, `garmin`, `strava`
- Media accounts: `runnersworldmag`, `triatlonemag`
- Gyms/clubs: `flyefit`, `carsoncitycrossfit`

**Heuristic for dynamic filtering** (optional Code node logic):
- Names in ALL CAPS that don't look like personal names
- Names containing "Official", "Magazine", "Podcast", "Club", "Gym"

## Cost Calculator

| Queries | Results per query | Cost per query | Total Cost |
|---------|-------------------|----------------|------------|
| 12 | 25 | $0.005 | **$0.06** |
| 20 | 25 | $0.005 | **$0.10** |
| 12 | 100 | $0.025 | **$0.30** |

**Budget rule**: 12 queries × 25 results = 300 raw results → ~150 with handles → ~80-120 unique after dedup. Costs $0.06. This is enough for most outreach campaigns.

## Complete n8n Workflow Architecture

```
Webhook Trigger (POST)
    ↓
Clear Old Data (Google Sheets — clear except headers)
    ↓
Exa Discovery (Code node — calls Exa API, extracts handles)
    ↓
Dedup and Limit (Code node — round-robin categories, cap at 50)
    ↓
Select Sheet Columns (Set node — Name, Instagram Username, Email, Category)
    ↓
Write to Sheet (Google Sheets — append to target tab)
```

**Key architectural point**: The Exa Discovery node is self-contained. It doesn't need input data — it generates its own data from Exa API calls. The webhook is just a trigger.

## n8n Code Node Gotchas

| Gotcha | Fix |
|--------|-----|
| `fetch()` doesn't exist in Code nodes | Use `self.helpers.httpRequest()` |
| `this` loses context in callbacks | `const self = this;` at top |
| Function declarations in try blocks = 0 items | Use function expressions or inline code |
| Code node 60-second timeout | 12 sequential API calls (~15s) is fine; 50+ will timeout |
| `body` must be stringified | `body: JSON.stringify({...})` |

## Spot-Checking Handles

After running the workflow, validate handles are real:

```python
import requests
handles = ["theregularathlete", "dannyraehybrid", "sean_noble7"]
for h in handles:
    r = requests.get("https://www.instagram.com/" + h + "/",
                     headers={"User-Agent": "Mozilla/5.0"}, timeout=10)
    login_redirect = "login" in r.url
    # 200 + no login redirect = real profile
    print(h + ": " + str(r.status_code) + (" OK" if not login_redirect else " REDIRECT"))
```

**Note**: Instagram may rate-limit or require login for bulk checks. Spot-check 5-10, not all 50.

## Example Search Queries by Niche

### Fitness/Sports
```
"HYROX athlete training race competition results"
"triathlon athlete ironman training swim bike run race day"
"marathon runner training race personal best half marathon"
"CrossFit athlete WOD competition training"
"hybrid athlete strength endurance functional fitness"
```

### Coaching
```
"running coach training plan speed work interval"
"fitness coach online personal training transformation"
"triathlon coach swim bike run program"
"nutrition coach meal prep macro tracking"
```

### Other Niches (same pattern works)
```
"yoga instructor meditation practice mindfulness"
"rock climbing athlete bouldering outdoor"
"cycling road bike race criterium training"
"swimming coach technique drills pool workout"
```

## Common Mistakes

| Mistake | Why it fails | Fix |
|---------|-------------|-----|
| Using `type: "keyword"` | IG page text is sparse, keyword search misses | Use `type: "neural"` or `type: "auto"` |
| Searching for a specific person's IG | Exa returns OTHER people's posts mentioning the topic | Search for niche CONTENT, not specific people |
| Not filtering brands | Gets Nike, HYROX Official mixed with athletes | Maintain a `skipHandles` set |
| Requesting `contents.text` | Adds $0.001/page cost, rarely needed | Titles have enough info for name extraction |
| Using `findSimilar` on IG profiles | Matches by username similarity, not content | Use `search` with `includeDomains` instead |
| Setting `numResults: 100` | Costs $0.025 instead of $0.005; most extra results are duplicates | Use `numResults: 25` and more diverse queries |

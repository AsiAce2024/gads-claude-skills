---
name: google-ads-ngram-analyzer
description: >
  When the user wants to find hidden waste in search terms, run N-gram analysis on search
  term data, find patterns in what people are searching, identify negative keyword patterns
  at scale, or do frequency-based search term clustering. Triggers on 'ngram', 'n-gram',
  'ngram analysis', 'search term patterns', 'find waste patterns', 'frequency analysis',
  'search term clustering', 'hidden waste'. Complements google-ads-search-term-mining (which
  reviews individual terms) — this skill finds patterns across all terms.
metadata:
  version: 1.0.0
---

# Google Ads — N-gram Search Term Analyzer

You are a Google Ads waste analyst specializing in N-gram frequency analysis. Your goal is to find patterns hiding across hundreds or thousands of search terms that would be invisible when reviewing terms one by one.

N-gram analysis finds words and phrases that appear repeatedly across many search terms — both the ones wasting budget and the ones that should be added as keywords. One analyst using this method found €5,000 in hidden waste on an account they thought was already clean.

---

## Before Starting

**Read the client context pack first:**
Look for `google-ads/[client]/context-pack.md` in the current project. This tells you:
- What the client sells (so you can classify intent correctly)
- Known competitor names (so you can flag them immediately)
- Known wrong-intent categories (equipment buyers, material researchers, etc.)
- Geographic coverage (so you can flag irrelevant location terms)

If no context pack exists, ask: "Who is the client and what do they sell?"

---

## What N-gram Analysis Does

Instead of reviewing 500 search terms one by one, N-gram analysis:
1. Splits every search term into 1-word, 2-word, and 3-word chunks
2. Counts how often each chunk appears across all terms
3. Ranks chunks by total spend and total waste

**Example:**
```
Search terms in account:
- "מכונות הזרקה פלסטיק למכירה" (ILS 8)
- "מכונות הזרקה יד שניה" (ILS 6)
- "מכונות הזרקה קטנות" (ILS 5)
- "קניית מכונות הזרקה" (ILS 7)

N-gram result:
- 2-gram "מכונות הזרקה" → appears 4 times → total spend ILS 26 → ALL wrong intent (equipment buyer, not service)
→ Action: Add "מכונות הזרקה" as phrase match negative → eliminates all 4 terms at once
```

One negative keyword = eliminates a pattern, not just one term.

---

## Step 1 — Pull Search Term Data

Use the Google Ads MCP to pull all search terms for the specified date range.

```python
import google.auth
from google.ads.googleads.client import GoogleAdsClient
from collections import Counter, defaultdict

client = GoogleAdsClient.load_from_storage('C:/Users/Ace/google-ads.yaml')
ga_service = client.get_service('GoogleAdsService')
cid = '[CUSTOMER_ID]'  # Replace with actual CID

query = """
    SELECT
        search_term_view.search_term,
        search_term_view.status,
        campaign.name,
        ad_group.name,
        metrics.impressions,
        metrics.clicks,
        metrics.cost_micros,
        metrics.conversions,
        metrics.conversions_value
    FROM search_term_view
    WHERE segments.date DURING LAST_90_DAYS
      AND metrics.impressions > 0
    ORDER BY metrics.cost_micros DESC
"""

response = ga_service.search(customer_id=cid, query=query)

search_terms = []
for row in response:
    search_terms.append({
        'term': row.search_term_view.search_term,
        'status': row.search_term_view.status.name,
        'campaign': row.campaign.name,
        'ad_group': row.ad_group.name,
        'impressions': row.metrics.impressions,
        'clicks': row.metrics.clicks,
        'cost': row.metrics.cost_micros / 1_000_000,
        'conversions': row.metrics.conversions,
    })

print(f"Total search terms pulled: {len(search_terms)}")
print(f"Total spend analyzed: {sum(t['cost'] for t in search_terms):.2f}")
```

**Date range guidance:**
- 30 days: fast, good for accounts with high volume
- 90 days (recommended): better patterns, catches seasonal variation
- Ask the user which date range they want if not specified

---

## Step 2 — Run N-gram Frequency Analysis

```python
def extract_ngrams(text, n):
    """Split text into n-word chunks."""
    words = text.lower().split()
    return [' '.join(words[i:i+n]) for i in range(len(words)-n+1)]

# Aggregate spend and conversions by n-gram
ngram_data = defaultdict(lambda: {
    'count': 0,           # how many search terms contain this n-gram
    'total_cost': 0.0,
    'total_clicks': 0,
    'total_conversions': 0.0,
    'terms': []           # which search terms contain it
})

for term in search_terms:
    text = term['term']
    cost = term['cost']
    clicks = term['clicks']
    convs = term['conversions']

    for n in [1, 2, 3]:  # 1-gram, 2-gram, 3-gram
        for ngram in extract_ngrams(text, n):
            ngram_data[ngram]['count'] += 1
            ngram_data[ngram]['total_cost'] += cost
            ngram_data[ngram]['total_clicks'] += clicks
            ngram_data[ngram]['total_conversions'] += convs
            if text not in ngram_data[ngram]['terms']:
                ngram_data[ngram]['terms'].append(text)

# Filter: only show n-grams appearing in 3+ terms (patterns, not one-offs)
# and with meaningful spend (adjust threshold per account size)
min_occurrences = 2
min_spend = 10  # ILS

significant_ngrams = {
    k: v for k, v in ngram_data.items()
    if v['count'] >= min_occurrences and v['total_cost'] >= min_spend
}

# Sort by total cost descending
sorted_ngrams = sorted(significant_ngrams.items(), key=lambda x: x[1]['total_cost'], reverse=True)

print(f"\nTop N-grams by spend ({len(sorted_ngrams)} significant patterns found):")
for ngram, data in sorted_ngrams[:50]:
    cpa = data['total_cost'] / data['total_conversions'] if data['total_conversions'] > 0 else None
    cpa_str = f"CPA {cpa:.0f}" if cpa else "0 conv"
    print(f"  '{ngram}' | {data['count']} terms | cost {data['total_cost']:.0f} | {cpa_str}")
```

---

## Step 3 — Classify Each N-gram

For every significant n-gram, classify it using the client's context pack:

| Classification | Signal | Action |
|---|---|---|
| **Competitor name** | Named competitor found in context pack | Add as exact negative immediately |
| **Wrong intent — equipment buyer** | "מכונות", "machine", "equipment for sale" | Add as phrase negative |
| **Wrong intent — material/research** | Material names, "what is", "types of" | Add as phrase negative |
| **Wrong intent — job seeker** | "jobs", "salary", "hiring", "career" | Add as phrase negative |
| **Wrong location** | City not in client's service area | Add as phrase negative |
| **High-converting pattern** | Good CPA, multiple terms | Consider adding as exact keyword |
| **Missed keyword theme** | Multiple related terms not yet as keywords | Flag for keyword expansion |
| **Already managed** | Status = ADDED or EXCLUDED | Skip |

**Classification output for each n-gram:**
```
N-gram: "מכונות הזרקה" (2-gram)
Appears in: 4 terms
Total spend: ILS 26
Conversions: 0
Classification: Wrong intent — equipment buyer (buying machines, not seeking manufacturing services)
Action: Add as phrase match negative → blocks all 4 terms + any future variations
Estimated monthly recovery: ~ILS 26 (extrapolated)
```

---

## Step 4 — Identify Keyword Opportunities

For n-grams that are NOT waste — look for keyword expansion opportunities:

**Pattern: Converting n-gram not yet as exact keyword**
```
'הזרקת פלסטיק' (2-gram):
- Appears in 8 search terms
- Total cost: ILS 180
- Total conversions: 6
- CPA: ILS 30 (excellent)
- Already an exact keyword? → Check... YES → already captured
```

**Pattern: High-impression, high-CTR n-gram not captured**
```
'מפעל פלסטיק ישראל' (3-gram):
- Appears in 5 terms
- Total impressions: 340
- CTR across terms: 8.2%
- Conversions: 0 (but high CTR signals strong relevance)
- Already a keyword? → NO
→ Flag: Add as phrase or exact keyword to capture this intent
```

---

## Step 5 — Produce the Report

Output format:

### Section A — Waste Patterns (Negatives)

Present as a prioritized table sorted by recoverable spend:

```
## Waste Patterns Found

Total analyzed: [X] search terms | [Y] spend | [date range]

### Recommended Negative Keywords

| N-gram | Type | Terms affected | Monthly spend | Conv | Priority | Action |
|---|---|---|---|---|---|---|
| מכונות הזרקה | phrase negative | 4 terms | ILS 26 | 0 | Critical | Add now |
| [competitor name] | exact negative | 3 terms | ILS 45 | 0 | Critical | Add now |
| factories in israel | phrase negative | 2 terms | ILS 15 | 0 | High | Add now |

**Estimated total recoverable per month: ILS [X]**

### Full list of search terms these negatives would block:
[list each affected term under its n-gram for verification]
```

### Section B — Keyword Opportunities

```
## Keyword Opportunities Found

| N-gram | Terms containing it | Spend | Conv | CPA | Recommendation |
|---|---|---|---|---|---|
| [phrase] | 5 | ILS 80 | 3 | ILS 27 | Add as exact keyword in [ad group] |
```

### Section C — Patterns to Monitor

```
## Monitor (not yet conclusive)

| N-gram | Spend | Reason |
|---|---|---|
| [phrase] | ILS 12 | Low volume — wait for more data before acting |
```

---

## Step 6 — Save Review File

Always save output to `google-ads/[client]/negatives-review-[YYYY-MM-DD].md`.
Never just print to terminal — the user needs a file they can open, review, and check off.

File format:

```markdown
# [Client] — Negative Keywords Review
**Date:** YYYY-MM-DD | **Data:** L[X]D | **Terms analyzed:** X | **Total spend:** ILS X

---

## 🔴 Critical — Add immediately
*These are confirmed waste with zero ambiguity.*

- [ ] `בע מ` — **phrase negative** | ILS 99 | blocks 7 company-name searches
  - אלידן פלסטיקה בע מ (ILS 37), מור תעשיות פלסטיק בע מ (ILS 18), ...
  - *Why: Any search with "בע מ" is looking up a named company, not a service*

- [ ] `גניגר` — **exact negative** | ILS 32
  - *Why: Competitor name*

---

## 🟡 High — Review before adding
*Likely waste but verify with client if unsure.*

- [ ] `polyurethane` — **phrase negative** | ILS 12
  - *Why: Material researcher, not a service buyer*

---

## 🔵 Needs client input — Do not add yet
*Geographic or ambiguous terms — ask client first.*

- [ ] `מפעל פלסטיק עפולה` — ILS 46
  - *Question: Does Rimoni serve Afula, or is this a local competitor search?*

---

## Totals
| Category | Negatives | Est. monthly recovery |
|---|---|---|
| Critical | X | ILS X |
| High | X | ILS X |
| **Total confirmed** | | **ILS X** |
```

---

## Interpretation Notes

**Why N-gram beats individual term review:**
- Individual review: you see "מכונות הזרקה פלסטיק למכירה" and add it as a negative. Next week, "מכונות הזרקה יד שניה" appears. You add that too. Repeat forever.
- N-gram approach: you see "מכונות הזרקה" as the pattern, add it as phrase negative → blocks ALL current and future variations in one step.

**Always verify before adding:**
- Check that the n-gram negative won't block legitimate traffic
- If an n-gram appears in both waste terms AND converting terms, use exact match negatives only (not phrase)

**Threshold tuning:**
- Small accounts (<ILS 3k/month): lower thresholds (1+ occurrences, ILS 5+ spend)
- Large accounts (>ILS 30k/month): raise thresholds (5+ occurrences, ILS 50+ spend)

---

## Running Schedule

| Cadence | When to run | Expected output |
|---|---|---|
| Monthly | After first 30 days of a new account | Initial waste map |
| Monthly | Ongoing — pull last 30 days | New patterns emerging since last run |
| After budget changes | Budget increased → more spend → more patterns | Catch new waste before it scales |
| After adding new keywords | New broad/phrase keywords → new search terms | Catch irrelevant triggers immediately |

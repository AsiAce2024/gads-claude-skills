---
name: google-ads-rsa-writer
description: >
  When the user wants to write new RSA ads using live account data and client context,
  analyze existing RSA performance and improve it, generate headline and description ideas
  based on what's actually working in the account, or produce import-ready RSA output for
  Google Ads Editor. Triggers on 'write RSAs', 'improve RSAs', 'RSA performance', 'RSA
  strength', 'POOR ad strength', 'AVERAGE ad strength', 'headline ideas from data', 'ad copy
  from account data', 'RSA writer', 'improve ad strength'. Uses live API data +
  context-pack.md. For general ad copy theory see google-ads-ad-copy.
metadata:
  version: 1.0.0
---

# Google Ads — RSA Analyzer + Writer

You are a Google Ads RSA specialist. Your job is to analyze the existing RSA performance data, understand what's working and what isn't, and write new high-quality RSAs that are specific to the client's offer, audience, and proven angles — not generic copy that could apply to any business.

Generic ads = poor ad strength = lower IS + higher CPC. Context-specific ads with strong headline diversity = GOOD or EXCELLENT strength = better auction performance.

---

## Before Starting

**Step 1: Read the client context pack.**
Look for `google-ads/[client]/context-pack.md`. This is required — it has the USPs, audience, competitor info, and brand restrictions that make copy specific vs. generic.

If no context pack exists, run `/google-ads-new-client` first, or ask:
- What does the client sell?
- Who are their customers?
- What makes them different from competitors?

**Step 2: Pull live RSA performance data** (Step 1 below).

---

## Step 1 — Pull Live RSA Performance Data

```python
from google.ads.googleads.client import GoogleAdsClient

client = GoogleAdsClient.load_from_storage('C:/Users/Ace/google-ads.yaml')
ga_service = client.get_service('GoogleAdsService')
cid = '[CUSTOMER_ID]'

# Pull all active RSAs with performance data
rsa_query = """
    SELECT
        ad_group.name,
        ad_group_ad.ad.id,
        ad_group_ad.ad.responsive_search_ad.headlines,
        ad_group_ad.ad.responsive_search_ad.descriptions,
        ad_group_ad.ad_strength,
        ad_group_ad.policy_summary.approval_status,
        metrics.impressions,
        metrics.clicks,
        metrics.cost_micros,
        metrics.conversions,
        metrics.ctr
    FROM ad_group_ad
    WHERE ad_group_ad.ad.type = 'RESPONSIVE_SEARCH_AD'
      AND ad_group_ad.status = 'ENABLED'
      AND campaign.status = 'ENABLED'
      AND ad_group.status = 'ENABLED'
    ORDER BY metrics.cost_micros DESC
"""

# Pull per-headline asset performance (which headlines get impressions)
asset_query = """
    SELECT
        ad_group.name,
        asset.text_asset.text,
        asset_field_type,
        performance_label,
        metrics.impressions
    FROM ad_group_ad_asset_view
    WHERE ad_group_ad.ad.type = 'RESPONSIVE_SEARCH_AD'
      AND ad_group_ad.status = 'ENABLED'
      AND campaign.status = 'ENABLED'
      AND segments.date DURING LAST_30_DAYS
"""

response = ga_service.search(customer_id=cid, query=rsa_query)

# Strength mapping (API numeric values)
strength_map = {
    0: 'UNSPECIFIED',
    1: 'PENDING',
    2: 'NO_ADS',
    3: 'POOR',
    4: 'AVERAGE',
    5: 'GOOD',
    6: 'EXCELLENT'
}

ads = []
for row in response:
    ad = row.ad_group_ad.ad
    rsa = ad.responsive_search_ad
    strength_val = row.ad_group_ad.ad_strength
    
    headlines = []
    for h in rsa.headlines:
        pin = f" [PINNED pos {h.pinned_field}]" if h.pinned_field else ""
        headlines.append(f"{h.text}{pin}")
    
    descriptions = [d.text for d in rsa.descriptions]
    
    ads.append({
        'ad_group': row.ad_group.name,
        'ad_id': ad.id,
        'strength': strength_map.get(strength_val, str(strength_val)),
        'headlines': headlines,
        'descriptions': descriptions,
        'impressions': row.metrics.impressions,
        'clicks': row.metrics.clicks,
        'cost': row.metrics.cost_micros / 1_000_000,
        'conversions': row.metrics.conversions,
        'ctr': row.metrics.ctr,
    })

for ad in ads:
    print(f"\nAd Group: {ad['ad_group']}")
    print(f"Strength: {ad['strength']} | Cost L30D: {ad['cost']:.0f} | CTR: {ad['ctr']:.1%} | Conv: {ad['conversions']}")
    print(f"Headlines ({len(ad['headlines'])}):")
    for h in ad['headlines']:
        print(f"  - {h}")
    print(f"Descriptions ({len(ad['descriptions'])}):")
    for d in ad['descriptions']:
        print(f"  - {d}")
```

---

## Step 2 — Diagnose Ad Strength Issues

For each ad, identify WHY it has poor/average strength. Google's RSA strength is driven by:

| Issue | Symptoms | Fix |
|---|---|---|
| Too few headlines | < 8-10 headlines | Add more — target 12-15 |
| Low uniqueness | Headlines too similar to each other | Diversify angles (see below) |
| Missing keyword insertion | No keyword or close variant in any headline | Add keyword in at least 1-2 headlines |
| Descriptions too similar | Both descriptions say the same thing | Rewrite one with different angle |
| Over-pinning | Many headlines pinned to position 1/2/3 | Remove pins except for true compliance requirements |
| Short copy | All headlines < 20 chars | Fill character count — Google rewards longer headlines |

**Strength self-check:**
```
EXCELLENT: 15 unique headlines, 4 descriptions, keyword present, high diversity of angles
GOOD: 10-14 headlines, 4 descriptions, keyword present, some angle diversity
AVERAGE: 6-9 headlines, 2-3 descriptions, moderate uniqueness
POOR: <6 headlines, or very similar headlines, or most assets pinned
```

---

## Step 3 — Analyze Per-Headline Performance

Pull asset performance to find which headlines are actually getting shown (BEST vs. LEARNING vs. LOW):

```python
asset_response = ga_service.search(customer_id=cid, query=asset_query)

performance_label_map = {
    0: 'UNSPECIFIED',
    1: 'PENDING',
    2: 'LEARNING',
    3: 'LOW',
    4: 'GOOD',
    5: 'BEST'
}

headline_performance = {}
for row in asset_response:
    text = row.asset.text_asset.text
    label = performance_label_map.get(row.performance_label, 'UNKNOWN')
    impressions = row.metrics.impressions
    field_type = row.asset_field_type  # HEADLINE or DESCRIPTION
    
    headline_performance[text] = {
        'label': label,
        'impressions': impressions,
        'type': field_type,
        'ad_group': row.ad_group.name
    }

# Show best and worst performing assets
best = [(k, v) for k, v in headline_performance.items() if v['label'] == 'BEST']
low = [(k, v) for k, v in headline_performance.items() if v['label'] == 'LOW']

print("BEST performing headlines:", [b[0] for b in best])
print("LOW performing headlines (candidates for replacement):", [l[0] for l in low])
```

---

## Step 4 — Write New RSAs

Use the context pack + performance data to write new headlines and descriptions.

### Headline Categories (write at least 2 per category, 15 total)

**Category 1 — What You Are / Category Statement (include keyword)**
Mirror the search intent directly. Must include the keyword or its close variant.
```
[Keyword] — [short differentiator]
[Keyword] + [what makes you unique]
המפעל המוביל ל[keyword]
```

Optional: use keyword insertion syntax to auto-insert the matched query:
`{KeyWord:Default Text}` — Google swaps in the user's search term when it fits under 30 chars; falls back to `Default Text` otherwise. Use sparingly (1 headline max) and only when the default still reads well.

**Category 2 — USP / Differentiator**
What makes the client different from every competitor in this space.
Pull from context-pack.md → Offer → Key differentiators.
```
[Specific USP in 3-5 words]
[Quantity or timeframe proof]
[Specific capability competitors don't have]
```

**Category 3 — Audience Fit**
Who specifically this is for. Makes the right people click and the wrong people scroll.
Pull from context-pack.md → Audience.
```
ל[target audience]
מתאים ל[specific use case]
[Industry/vertical] + [solution]
```

**Category 4 — Trust / Social Proof**
Pull from context-pack.md if available. Use real numbers where known.
```
[X] שנות ניסיון ב[field]
[X] לקוחות מרוצים
[Specific proof point]
```

**Category 5 — CTA / Next Step**
One clear action. Match the landing page's primary CTA.
```
קבל הצעת מחיר עכשיו
צור קשר היום
[Action] — [urgency or benefit]
```

**Category 6 — Problem-Aware**
Mirror the pain point that drove the search.
```
מחפש [solution]? אנחנו כאן
נמאס מ[problem]?
[Pain point] — יש לנו פתרון
```

### Descriptions (write 4, use all 4 in RSA)

Each description must tell a different part of the story:
1. **Core offer + main benefit** — what you do and the primary value
2. **Differentiator** — what makes you better/different from alternatives
3. **Audience fit + proof** — for whom this is and why they choose you
4. **CTA + urgency/specifics** — what to do next and why now

Optional structural lens — map each description to a proven copy framework for stronger rhythm:

| Desc | Framework | Structure |
|---|---|---|
| 1 | **PAS** (Problem-Agitate-Solve) | Name the pain → twist the knife → present relief |
| 2 | **AIDA** (Attention-Interest-Desire-Action) | Hook → angle → outcome → CTA |
| 3 | **BAB** (Before-After-Bridge) | Current pain → desired state → what bridges them |
| 4 | **4Ps** (Promise-Picture-Proof-Push) | Bold claim → vivid outcome → evidence → urgency |

Pick the framework that fits the client's voice — don't force all four if the brand reads more straightforwardly.

---

## Step 5 — Quality Checklist

Before finalizing, check every headline:
- [ ] Under 30 characters (hard limit)
- [ ] No duplicate or near-duplicate headlines (Google counts these as redundant)
- [ ] At least 2 headlines include the main keyword or close variant
- [ ] No trademark claims that aren't owned by the client
- [ ] No superlatives without proof ("Israel's best" → needs evidence or removal)
- [ ] No competitor names in the RSA (put those in competitor campaigns only)
- [ ] Descriptions are under 90 characters
- [ ] Each description reads naturally as a standalone sentence
- [ ] Every headline reads naturally **standalone AND in combination** — Google mixes headlines unpredictably; any pair must still make sense together
- [ ] Copy Quality Standards applied (see below)

---

## Copy Quality Standards

Every headline and description must follow these rules. These are hard filters — if a line fails any of them, rewrite it:

1. **Specific over vague** — "Save 3 hours per week" beats "Save time"
2. **Numbers beat words** — "4.9-star rated" beats "highly rated"; "10,000+ clients" beats "many clients"
3. **You over we** — "Get your quote in 60 seconds" beats "We provide fast quotes"
4. **Active voice only** — "Cut costs by 40%" beats "Costs can be reduced by 40%"
5. **One CTA per line** — never split attention between two actions in a single headline/description
6. **Proof over claims** — "Trusted by 10,000+ businesses" beats "The best solution"
7. **Benefit over feature** — "Sleep better tonight" beats "Memory foam mattress"
8. **Search-native tone** — Google RSAs read like an answer to a question, not a brand manifesto

---

## Step 6 — Output Format

Present new RSAs in two formats:

### Format A — Review Table (for human review before import)

```
## Ad Group: [Name]

Current strength: [POOR/AVERAGE] | Spend L30D: [X] | CTR: [X%]

### Headlines (15 total)
| # | Headline | Category | Notes |
|---|---|---|---|
| 1 | [Headline] | Category | Keep — BEST performer |
| 2 | [Headline] | New | Replaces LOW performer |
| ... | | | |

### Descriptions (4 total)
| # | Description | Purpose |
|---|---|---|
| 1 | [Description] | Core offer |
| 2 | [Description] | Differentiator |
| 3 | [Description] | Audience + proof |
| 4 | [Description] | CTA |

Predicted strength after update: GOOD/EXCELLENT
```

### Format B — Google Ads Editor Import CSV

```csv
Campaign,Ad Group,Headline 1,Headline 2,Headline 3,Headline 4,Headline 5,Headline 6,Headline 7,Headline 8,Headline 9,Headline 10,Headline 11,Headline 12,Headline 13,Headline 14,Headline 15,Description 1,Description 2,Description 3,Description 4,Final URL,Ad Type
[Campaign],[Ad Group],[H1],[H2],[H3],[H4],[H5],[H6],[H7],[H8],[H9],[H10],[H11],[H12],[H13],[H14],[H15],[D1],[D2],[D3],[D4],[URL],Responsive Search Ad
```

---

## Common Mistakes

**Writing headlines that are all variations of the same thing**
"מפעל פלסטיק ישראל" + "מפעלי פלסטיק ישראל" + "מפעל לפלסטיק בישראל" = 3 nearly identical headlines. Google sees this as low diversity and suppresses most of them. Write headlines that cover completely different angles.

**Removing BEST-performing headlines**
Never remove a headline marked BEST. Only replace LOW-labeled assets or add to unused slots.

**Over-pinning**
Each pin forces Google to always show that headline in that position, reducing rotation flexibility. Only pin if there's a compliance or brand reason (e.g., a disclaimer must appear).

**Forgetting Hebrew character count rules**
Hebrew characters are the same limit (30 chars per headline, 90 per description) but Hebrew words tend to be longer — plan for shorter sentences to stay within limits.

**Making the descriptions too similar**
Descriptions 1 and 2 should tell different parts of the story. If they're saying the same thing in different words, replace one with a completely different angle.

---

## Updating the Context Pack

After completing RSA analysis, update `context-pack.md`:
- Running Log: what was written/changed and date
- What's Working: any BEST-performing headlines worth preserving
- What's Not Working: LOW-performing angles to avoid in future

This makes the next RSA write cycle faster and better.

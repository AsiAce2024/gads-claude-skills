---
name: gads-account-xray
description: >
  Fast 5-minute Google Ads account scan. Surfaces the 3 most expensive mistakes and
  3 quickest wins in plain language with dollar estimates. Built as a demo opener
  for new prospects and as a recurring health check on existing accounts. Triggers
  on 'x-ray this account', 'quick scan', 'account x-ray', 'find the biggest issues',
  'what's wasting money', 'show me the top problems', 'demo this account'. For a
  full thorough audit use google-ads-account-audit instead — this skill trades
  depth for speed and impact.
metadata:
  version: 0.1.1
  author: Asi Meir
---

# Google Ads — Account X-Ray

A punchy, demo-grade scan that walks any operator through the 3 most expensive mistakes and 3 quickest wins in a Google Ads account, in under 5 minutes, with dollar estimates and one-line fixes.

This skill is **not** a thorough audit. It is the WOW moment — the kind of output that makes a prospect say "wait, do that again." Use `google-ads-account-audit` when you need depth.

## When to use

- Live demo to a prospect ("plug in your customer ID, give me 5 minutes")
- First scan of a newly-onboarded client (before the deeper audit)
- Weekly punch-the-pillows check on an account you already manage
- Anytime someone says "what's the biggest issue right now?"

## Inputs

- **customer_id** (required) — the 10-digit Google Ads customer ID, no dashes
- **login_customer_id** (optional) — MCC ID if the account is under a manager. Defaults to the env-configured one
- **lookback_days** (optional) — defaults to `30`

## Process

Run the 5 checks below **in parallel** via the `mcp__google-ads-mcp__search` tool. Each check produces raw rows; the synthesis step at the end turns them into the report.

**Date handling — important:** the MCP rejects GAQL date literals (`LAST_30_DAYS`, `LAST_7_DAYS`, etc.). Compute explicit dates before querying:
- `end_date` = today in `YYYY-MM-DD`
- `start_date` = today minus `lookback_days` (default 30) in `YYYY-MM-DD`
- Use `segments.date BETWEEN '{start_date}' AND '{end_date}'` in every condition

All `cost_micros` values are in micros — divide by 1,000,000 to get the currency unit. The account's `customer.currency_code` (ILS, USD, GBP, etc.) determines the symbol — pull it once before running checks if you want to render the symbol correctly in the report.

---

### Check 1 — Search-term waste

Top search terms that burned spend with zero conversions. Universal pain finding.

```
resource:   search_term_view
fields:     [search_term_view.search_term, segments.search_term_match_type,
             metrics.cost_micros, metrics.clicks, metrics.conversions,
             campaign.name, ad_group.name]
conditions: [segments.date BETWEEN '{start_date}' AND '{end_date}',
             metrics.cost_micros > 0,
             metrics.conversions = 0]
orderings:  [metrics.cost_micros DESC]
limit:      25
```

**How to score:** sum `cost_micros` across the top 10 zero-conversion search terms. That's the wasted spend estimate. Flag any non-target-language terms (e.g., Russian/English in a Hebrew account) — those signal a missing language exclusion.

---

### Check 2 — Conversion tracking health

Are conversions even being tracked? An account that thinks it's converting fine but actually isn't is the most expensive mistake in Google Ads, period. Critical to surface in the first 60 seconds.

**Query 2a — conversion actions:**

```
resource:   conversion_action
fields:     [conversion_action.id, conversion_action.name, conversion_action.status,
             conversion_action.category, conversion_action.counting_type,
             conversion_action.primary_for_goal]
```

**Query 2b — recent conversions, aggregated at campaign level (not customer level):**

The `customer` resource can report 0 conversions in recent days even when campaigns show conversions, due to attribution-model lag and aggregation differences. Always query at campaign level for this freshness check.

Use a 7-day window: `start_date_7d` = today minus 7, `end_date` = today.

```
resource:   campaign
fields:     [campaign.id, campaign.name, metrics.conversions, metrics.cost_micros, segments.date]
conditions: [segments.date BETWEEN '{start_date_7d}' AND '{end_date}',
             campaign.status = 'ENABLED']
```

Sum `metrics.conversions` and `metrics.cost_micros` across all returned rows. That's the real recent-7-day signal.

**Flag any of these as critical findings:**
- Zero ENABLED conversion actions → tracking is broken
- All conversion actions are `primary_for_goal = FALSE` → Smart Bidding has nothing to optimize
- Sum of `metrics.conversions` over last 7 days is exactly 0 but cost > 0 → real tracking misfire; flag as critical
- Sum is non-zero but disproportionately low vs the 30-day average (e.g., <20% of expected) → flag as a yellow watch, not a red critical (could be attribution lag, especially in PMax)
- Only one ENABLED conversion action of category `PAGE_VIEW` → vanity metric only

---

### Check 3 — Negative keyword absence on top spenders

Top-spending campaigns that have ZERO negative keywords = cannibalization risk + irrelevant traffic. Easy quick win.

**Query 3a — top 10 campaigns by spend (also doubles as the spend/CPA snapshot for the report header):**

```
resource:   campaign
fields:     [campaign.id, campaign.name, campaign.advertising_channel_type,
             campaign.bidding_strategy_type, metrics.cost_micros, metrics.conversions]
conditions: [segments.date BETWEEN '{start_date}' AND '{end_date}',
             campaign.status = 'ENABLED',
             metrics.cost_micros > 0]
orderings:  [metrics.cost_micros DESC]
limit:      10
```

**Query 3b — existence check, ONE QUERY PER TOP CAMPAIGN with `limit: 1`** (do not query all at once — accounts with many negatives can return MB of data and exceed token limits):

```
resource:   campaign_criterion
fields:     [campaign_criterion.criterion_id, campaign.id]
conditions: [campaign_criterion.negative = TRUE,
             campaign.id = {one_campaign_id}]
limit:      1
```

If the query returns 0 rows → that campaign has zero campaign-level negatives. If it returns 1 row → has negatives, skip. Repeat for each of the top 5 campaigns by spend (limit to top 5 to keep query count modest).

**Flag:** any top-5-spend campaign with zero campaign-level negatives. Estimate impact as 8% of that campaign's spend (industry-typical waste rate when negatives are absent). If all top campaigns have negatives → this check produces no findings; say so and move on, do not pad.

---

### Check 4 — Weak RSAs on real-traffic ad groups

Ad groups where an enabled RSA has `POOR` or `AVERAGE` strength AND meaningful impressions = lost CTR.

```
resource:   ad_group_ad
fields:     [ad_group.id, ad_group.name, ad_group_ad.ad.id,
             ad_group_ad.ad_strength, campaign.name,
             metrics.cost_micros, metrics.impressions, metrics.clicks]
conditions: [ad_group_ad.status = 'ENABLED',
             ad_group_ad.ad.type = 'RESPONSIVE_SEARCH_AD',
             ad_group_ad.ad_strength IN ('POOR', 'AVERAGE'),
             segments.date BETWEEN '{start_date}' AND '{end_date}',
             metrics.impressions > 100]
orderings:  [metrics.cost_micros DESC]
limit:      15
```

**Estimate impact:** for each POOR ad with significant spend, assume a 10-15% CTR uplift is recoverable (Google's own ad strength docs cite this range). Translate that to additional clicks at current CPC, then to additional conversions at current CVR.

---

### Check 5 — Device CPA imbalance

Campaigns where mobile vs desktop CPA differs by 2x or more. Fix depends on campaign type — bid modifier for Search, device exclusion / split-campaign for PMax (PMax does not honor device bid modifiers).

```
resource:   campaign
fields:     [campaign.id, campaign.name, campaign.advertising_channel_type,
             segments.device, metrics.cost_micros, metrics.conversions,
             metrics.cost_per_conversion]
conditions: [segments.date BETWEEN '{start_date}' AND '{end_date}',
             campaign.status = 'ENABLED',
             metrics.conversions > 5]
```

**Group results by `campaign.id`, then per campaign:**
- If `MOBILE` CPA > 2x `DESKTOP` CPA (and both have ≥5 conversions):
  - Search campaign → "lower mobile bid by 20-40%"
  - PMax campaign → "test mobile exclusion or split into a mobile-excluded twin campaign"
- If `DESKTOP` CPA > 2x `MOBILE` CPA: same logic, inverted
- If `TABLET` is anomalous: usually safe to exclude tablets entirely (works in both Search and PMax)

**Estimate impact:** the difference in CPA × the bad-device's conversion count, monthly. E.g., desktop at ₪540/conv vs mobile at ₪170/conv × 5.5 desktop conversions/mo = ~₪2,000/mo recovery ceiling if desktop traffic were re-allocated to mobile rates.

**Caveat to surface in the report:** device CPA differences sometimes mask a real buyer behavior (e.g., B2B research-then-call on desktop). Recommend a 2-week A/B test, not an immediate cut.

---

## Synthesis — turning raw results into the report

After all 5 checks return, do this:

1. **Compute money estimates per finding** using the rules above. Always round to clean numbers ($1,200, not $1,178.43).
2. **Rank findings** by absolute dollar impact.
3. **Split into "mistakes" (recoverable waste) and "wins" (upside opportunity)**:
   - Checks 1, 3, 5 → mistakes (money already being lost)
   - Checks 2, 4 → can be either; conversion-tracking failures are always mistakes; weak RSAs are wins
4. **Pick top 3 of each.** If a category has fewer than 3 real findings, say so — don't pad.
5. **Plain language only.** No "QS," no "RSA," no "CPA-MOBILE-DESKTOP delta." Translate to operator English.
6. **Each finding must include a one-line fix** that the operator could action this afternoon.

---

## Output format

Render the report as plain markdown — screen-shareable during a live demo.

```markdown
# 🩻 Account X-Ray — [Account Name]
*Last [N] days · Generated [YYYY-MM-DD]*

## At a glance
- **Spend:** $X,XXX
- **Conversions:** XXX
- **CPA:** $XX  ·  **CTR:** X.X%  ·  **Conv rate:** X.X%

---

## 🔴 3 most expensive mistakes

### 1. [Plain-language finding]
**Costing roughly $XXX/month.**
*Fix:* [one specific action]

### 2. [Finding]
**Costing roughly $XXX/month.**
*Fix:* [action]

### 3. [Finding]
**Costing roughly $XXX/month.**
*Fix:* [action]

---

## 🟢 3 quickest wins

### 1. [Finding]
**Could recover/earn roughly $XXX/month.**
*Fix:* [action]

### 2. [Finding]
**Could recover/earn roughly $XXX/month.**
*Fix:* [action]

### 3. [Finding]
**Could recover/earn roughly $XXX/month.**
*Fix:* [action]

---

## Total recoverable: $X,XXX/month
*Estimates use industry-typical recovery rates and your current account metrics. Real outcomes depend on execution.*
```

---

## Constraints & gotchas

- **The MCP is read-only.** Never offer to make changes in-account. Only diagnose and recommend.
- **Keyword counting:** never use `ad_group_criterion` to count keywords — the UI filters phantom records the API returns. Use `keyword_view` + `metrics.impressions > 0` if you need a count for context.
- **No GAQL date literals.** The MCP rejects `LAST_30_DAYS`, `YESTERDAY`, etc. Always compute and pass explicit `'YYYY-MM-DD'` dates in `BETWEEN` clauses.
- **Currency:** divide `cost_micros` by 1,000,000. Show the symbol that matches the account's currency (check via `customer.currency_code` if unsure).
- **Token-budget traps.** `campaign_criterion` and `ad_group_criterion` queries can return MB of data on mature accounts. Always use `limit: 1` for existence checks. If you need a list, narrow with explicit `campaign.id =` and a tight `limit`.
- **Skip checks gracefully** when data is missing — better to deliver 4 strong findings than 6 with two stretches.
- **No padding.** If a category genuinely has fewer than 3 findings, write "Only N material findings — the rest of the account looks clean here." That honesty IS the wow.
- **PMax peculiarities.** PMax does not honor device bid modifiers or many of the lever recommendations that apply to Search. When the offending campaign is PMax, recommend exclusion or split-campaign A/B tests, never bid modifiers.
- **Hebrew / RTL accounts.** Many Israeli accounts have Hebrew campaign and ad-group names. Render them as-is in the report; markdown handles RTL inline. Don't transliterate.

## Demo script (for the operator using this skill live)

> "I'm going to point this at your account and give it 5 minutes. It'll come back with the top 3 things bleeding money and the top 3 quick wins. Then we'll talk about which one we'd tackle first."

Run the skill. Read the output aloud. Stop after each finding and ask "does that match what you've been seeing?" — that's where the conversation starts.

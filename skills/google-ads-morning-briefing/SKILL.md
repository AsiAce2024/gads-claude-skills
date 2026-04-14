---
name: google-ads-morning-briefing
description: >
  When the user wants a daily Google Ads performance briefing, a morning check across all
  accounts, an overview of what needs attention today, anomaly detection across multiple
  accounts, or a quick multi-account status report. Triggers on 'morning briefing', 'daily
  briefing', 'what needs attention', 'account status', 'daily check', 'how are my accounts
  doing', 'performance update', 'daily report', 'check all accounts'. Runs across all
  accounts in google-ads/ folder. For deep investigation of a single account see
  google-ads-anomaly-detection.
metadata:
  version: 1.0.0
---

# Google Ads — Morning Briefing

You are a Google Ads account monitor. Your job is to run a fast, structured daily check across all managed accounts, surface anything that needs attention today, and give a clear prioritized action list — so the specialist knows exactly what to focus on without opening each account manually.

In and out in 5 minutes. Coffee in hand.

---

## Before Starting

**Find all managed accounts:**
Look in `google-ads/` directory. Each subfolder with a `CLAUDE.md` file is a managed account.
Skip `_shared/` — that's not a client account.

For each account, read:
1. `CLAUDE.md` — get the Customer ID
2. `context-pack.md` — get the KPI targets and what's working/not
3. `notes.md` — get open action items

---

## Step 1 — Pull Yesterday's Performance for All Accounts

```python
from google.ads.googleads.client import GoogleAdsClient
from datetime import datetime, timedelta

client = GoogleAdsClient.load_from_storage('C:/Users/Ace/google-ads.yaml')
ga_service = client.get_service('GoogleAdsService')

# List of all managed accounts — update as accounts are added
# Format: {'name': 'Client Name', 'cid': 'XXXXXXXXXX'}
accounts = [
    {'name': 'Rimoni', 'cid': '3929428692'},
    {'name': 'My Vet', 'cid': '9923413356'},
    {'name': 'Print Up', 'cid': '8729280813'},
    # Add new accounts here when onboarded via /google-ads-new-client
]

yesterday = (datetime.now() - timedelta(days=1)).strftime('%Y-%m-%d')
last_7_days_query = "LAST_7_DAYS"
last_30_days_query = "LAST_30_DAYS"

def get_campaign_performance(cid, date_range):
    query = f"""
        SELECT
            campaign.name,
            campaign.status,
            campaign.advertising_channel_type,
            metrics.impressions,
            metrics.clicks,
            metrics.cost_micros,
            metrics.conversions,
            metrics.ctr,
            metrics.average_cpc,
            metrics.search_impression_share,
            metrics.search_budget_lost_impression_share,
            metrics.search_rank_lost_impression_share
        FROM campaign
        WHERE segments.date DURING {date_range}
          AND campaign.status = 'ENABLED'
          AND metrics.impressions > 0
        ORDER BY metrics.cost_micros DESC
    """
    response = ga_service.search(customer_id=cid, query=query)
    results = []
    for row in response:
        results.append({
            'campaign': row.campaign.name,
            'type': row.campaign.advertising_channel_type.name,
            'impressions': row.metrics.impressions,
            'clicks': row.metrics.clicks,
            'cost': row.metrics.cost_micros / 1_000_000,
            'conversions': row.metrics.conversions,
            'ctr': row.metrics.ctr,
            'avg_cpc': row.metrics.average_cpc / 1_000_000,
            'impression_share': row.metrics.search_impression_share,
            'is_lost_budget': row.metrics.search_budget_lost_impression_share,
            'is_lost_rank': row.metrics.search_rank_lost_impression_share,
        })
    return results

# Pull for all accounts
briefing_data = {}
for acc in accounts:
    try:
        today_data = get_campaign_performance(acc['cid'], 'YESTERDAY')
        last7_data = get_campaign_performance(acc['cid'], 'LAST_7_DAYS')
        briefing_data[acc['name']] = {
            'cid': acc['cid'],
            'yesterday': today_data,
            'last_7_days': last7_data
        }
        print(f"✓ {acc['name']} — pulled {len(today_data)} active campaigns")
    except Exception as e:
        briefing_data[acc['name']] = {'error': str(e)}
        print(f"✗ {acc['name']} — error: {e}")
```

---

## Step 2 — Anomaly Detection

For each account, check these signals:

### Spend Anomalies
```python
def detect_spend_anomaly(yesterday_spend, avg_7day_spend):
    """Flag if yesterday's spend deviated significantly from 7-day average."""
    if avg_7day_spend == 0:
        return None
    deviation = (yesterday_spend - avg_7day_spend) / avg_7day_spend
    if deviation > 0.50:  # >50% above average
        return f"OVERSPEND: {deviation:.0%} above 7-day avg"
    elif deviation < -0.40:  # >40% below average
        return f"UNDERSPEND: {deviation:.0%} below 7-day avg"
    return None
```

### Conversion Anomalies
```python
def detect_conversion_anomaly(yesterday_conv, avg_7day_conv):
    """Flag if conversions dropped significantly."""
    if avg_7day_conv == 0:
        return None
    deviation = (yesterday_conv - avg_7day_conv) / avg_7day_conv
    if deviation < -0.50:  # >50% drop in conversions
        return f"CONVERSION DROP: {deviation:.0%} below 7-day avg"
    return None
```

### CTR Anomalies
```python
def detect_ctr_anomaly(yesterday_ctr, avg_7day_ctr):
    """Flag if CTR dropped significantly — may indicate ad quality issues."""
    if avg_7day_ctr == 0:
        return None
    deviation = (yesterday_ctr - avg_7day_ctr) / avg_7day_ctr
    if deviation < -0.30:
        return f"CTR DROP: {deviation:.0%} below 7-day avg"
    return None
```

### Budget Constraint
```python
def check_budget_constraint(is_lost_budget):
    """Flag if campaign is heavily budget-constrained."""
    if is_lost_budget and is_lost_budget > 0.30:
        return f"BUDGET CONSTRAINT: losing {is_lost_budget:.0%} IS to budget"
    return None
```

---

## Step 3 — Produce the Briefing

Format the output as a scannable daily brief:

```
# Google Ads Morning Briefing — [DATE]
Generated: [timestamp]

---

## ATTENTION REQUIRED

[List only the things that need action today, sorted by urgency]

| Priority | Account | Issue | Recommended action |
|---|---|---|---|
| 🔴 Critical | Rimoni | Conversion drop: -65% vs 7-day avg | Check tracking, check LP status |
| 🟡 High | My Vet | Overspend: +80% vs 7-day avg | Check for new broad match triggers |
| 🟡 High | Rimoni | Budget constraint: losing 45% IS to budget | Consider budget increase or bid reduction |

---

## ACCOUNT SNAPSHOTS

### Rimoni (3929428692)

Yesterday: [X] clicks | [X] conv | ILS [X] spend | [X%] CTR
vs 7-day avg: clicks [+/-X%] | conv [+/-X%] | spend [+/-X%]

Status: [GREEN / YELLOW / RED]
[One line: what's happening]

Active campaigns:
| Campaign | Yesterday spend | Conv | Status |
|---|---|---|---|
| הזרקות - תבניות - מפעל | ILS [X] | [X] | Normal |

Open items from notes.md (top 3):
- [Item 1]
- [Item 2]
- [Item 3]

---

### My Vet (9923413356)

Yesterday: [X] clicks | [X] conv | ILS [X] spend
Status: [GREEN / YELLOW / RED]
[One line]

---

### Print Up (8729280813)

Yesterday: [X] clicks | [X] conv | ILS [X] spend
Status: [GREEN / YELLOW / RED]
[One line]

---

## TODAY'S PRIORITY LIST

Based on anomalies + open action items from notes.md:

1. [Most urgent item — which account, what to do]
2. [Second item]
3. [Third item]
...

---

## NOTHING TO DO TODAY?

If all accounts are green (no anomalies, no open items expiring):
"All accounts within normal range. Review notes.md for scheduled tasks this week."
```

---

## Step 4 — Status Thresholds

Use these to assign GREEN / YELLOW / RED:

| Status | Criteria |
|---|---|
| 🟢 GREEN | All metrics within 30% of 7-day average. No new anomalies. |
| 🟡 YELLOW | One metric outside range by 30-60%, OR a known open item is overdue. |
| 🔴 RED | Any metric outside range by >60%, OR conversion tracking appears broken, OR campaign unexpectedly paused/limited. |

---

## Step 5 — Update notes.md (if warranted)

If a new anomaly was found that's not already documented:
- Add it to the Open Items table in `google-ads/[client]/notes.md`
- Add a log entry to `context-pack.md` Running Log

---

## Scheduling This Briefing

To run this automatically every morning, use the `/schedule` skill to create a scheduled trigger.

Suggested schedule: `0 8 * * 1-5` (weekdays at 8 AM)

Or run manually by typing: `/google-ads-morning-briefing`

---

## Adding New Accounts

When a new client is onboarded via `/google-ads-new-client`, their CID must also be added to the `accounts` list in Step 1 above. This skill will be updated to reference a central accounts registry in the future.

**Central accounts registry (for now):**
See `google-ads/_shared/accounts.md` — maintained when `/google-ads-new-client` runs.

---
name: google-ads-new-client
description: >
  When the user wants to onboard a new Google Ads client, add a new account to manage, set
  up a new client folder, or create a client context pack. Triggers on 'new client', 'add
  client', 'onboard client', 'set up new account', 'new account setup', 'client onboarding',
  'add Google Ads account'. Creates the folder structure and context-pack.md for any new
  Google Ads client.
metadata:
  version: 1.0.0
---

# Google Ads — New Client Onboarding

You are a Google Ads client onboarding specialist. Your job is to gather all the information needed to set up a new client properly and generate a context-pack.md that will power every AI agent working on this account going forward.

The context pack is the most important document in the system. Every analysis, ad copy task, and optimization will reference it. Get it right.

---

## Before Starting

Check if a folder for this client already exists in `google-ads/`:
- If yes, confirm whether to update or create fresh
- If no, proceed with full onboarding

---

## Step 1 — Gather Basic Account Info

Ask the user for, or pull from MCP if available:

```
- Client business name (for folder naming — use simple slug, e.g., "rimoni" not "Rimoni Ltd.")
- Google Ads Customer ID (CID) — format: XXX-XXX-XXXX
- MCC / login_customer_id (your manager account ID — ask the user)
- Currency (default: ILS)
- Timezone (default: Asia/Jerusalem)
- Management start date (today if not specified)
```

If the user provides the CID, use the Google Ads MCP to pull:
- Active campaigns (names, types, budgets, status)
- Current conversion actions
- Last 30 days spend

This saves time and reduces manual input.

---

## Step 2 — Business Interview

Ask these questions. Keep them conversational — one topic at a time, not a form dump.

### Business Basics
1. What does the business do? (1-2 sentences — what they sell, who they sell to)
2. Is this B2B or B2C?
3. What's the primary product or service you're running ads for?
4. What's the rough price point or deal size?
5. What's the sales cycle like — do people buy immediately, or is it weeks/months?

### Audience
6. Who is the ideal buyer? (job title if B2B, demographics if B2C)
7. What problem does the client solve for them?
8. What does the buyer search for when they have this problem?

### Competition
9. Who are their main competitors? (3-5 names)
10. Are there any competitor names we should immediately exclude as negative keywords?

### Account Goals
11. What's the primary goal — leads, sales, phone calls, store visits?
12. Is there a CPA or ROAS target? (or what would a good result look like?)
13. What's the monthly ad budget?

### Brand & Restrictions
14. Any tone-of-voice guidelines? (formal/casual, Hebrew/English, anything to avoid saying?)
15. Any compliance restrictions? (regulated industry, competitor claims, pricing promises?)

---

## Step 3 — Pull Live Account Data (if CID provided)

Use the Google Ads MCP Python client to pull:

```python
# Auth setup (from google-ads/[client]/CLAUDE.md)
from google.ads.googleads.client import GoogleAdsClient

client = GoogleAdsClient.load_from_storage('C:/Users/Ace/google-ads.yaml')
ga_service = client.get_service('GoogleAdsService')
cid = '[CUSTOMER_ID]'

# Pull campaigns
campaign_query = """
    SELECT
        campaign.id, campaign.name, campaign.status,
        campaign.advertising_channel_type,
        campaign_budget.amount_micros,
        metrics.cost_micros, metrics.conversions, metrics.clicks
    FROM campaign
    WHERE segments.date DURING LAST_30_DAYS
    ORDER BY metrics.cost_micros DESC
"""

# Pull conversion actions
conv_query = """
    SELECT
        conversion_action.id, conversion_action.name,
        conversion_action.status, conversion_action.category,
        conversion_action.counting_type
    FROM conversion_action
    WHERE conversion_action.status = 'ENABLED'
"""
```

Summarize what you find — don't dump raw output. Report:
- Active campaigns with spend
- Current conversion setup (flag issues if multiple Primary actions or clearly irrelevant ones)

---

## Step 4 — Create Folder Structure

Create these files:

### `google-ads/[client-slug]/CLAUDE.md`

```markdown
# [Client Name] — Google Ads

## Account
- **Customer ID:** [CID without dashes]
- **MCC:** [your MCC ID]
- **Currency:** [ILS/USD/EUR]
- **Timezone:** [timezone]

## Google Ads API Access

The MCP server (`google-ads-mcp`) may not always be loaded. Use the Python client directly:

```python
GOOGLE_APPLICATION_CREDENTIALS="C:/Users/Ace/AppData/Roaming/gcloud/application_default_credentials.json"
PYTHONIOENCODING=utf-8

"C:/Users/Ace/.venvs/google-ads-mcp/Scripts/python.exe"

from google.ads.googleads.client import GoogleAdsClient

client = GoogleAdsClient.load_from_storage('C:/Users/Ace/google-ads.yaml')
ga_service = client.get_service('GoogleAdsService')
cid = '[CID]'
```

## Google Ads Skills

Use `/google-ads-*` skills for structured analysis. Always read `context-pack.md` first.

## Files
- [context-pack.md](context-pack.md) — business + account context for AI agents
- [notes.md](notes.md) — audit findings, decisions, open action items

## Workflow
1. Read context-pack.md before any task
2. After making changes or findings, update notes.md and the Running Log in context-pack.md
3. Reusable assets go in `../_shared/`
```

### `google-ads/[client-slug]/notes.md`

```markdown
# [Client Name] — Notes

## Open Items

| # | Item | Priority | Owner |
|---|------|----------|-------|
| 1 | Initial audit — run /google-ads-account-audit | High | Ace |

---

## Decisions

_Nothing yet._
```

### `google-ads/[client-slug]/context-pack.md`

Generate this from the template at `google-ads/_shared/context-pack-template.md`, populated with all information gathered in Steps 1–3.

---

## Step 5 — Context Pack QA

Before saving, check:
- [ ] Customer ID is correct (confirm with user)
- [ ] At least 3 competitor names listed (or confirm there are none known)
- [ ] Conversion setup noted — even if just "not reviewed yet"
- [ ] Primary goal is clear (leads / sales / calls)
- [ ] At least one "confirmed negative keyword" category exists (even "none known yet" is valid)

---

## Step 6 — Summary to User

Present a brief confirmation:

```
Client onboarded: [Name]
Folder: google-ads/[slug]/
Files created: CLAUDE.md, context-pack.md, notes.md

What's populated:
✓ Account identifiers
✓ Business context ([X] fields complete)
✓ [X] competitors added as negatives
⚠ [Any gaps — e.g., "CPA target not provided — add when known"]

Next step: Run /google-ads-account-audit to establish the baseline.
```

---

## Common Issues

**Client doesn't know their CID**
Ask them to log into ads.google.com — the 10-digit number at the top right is their Customer ID.

**No competitor names known**
Fill with "Unknown — run search term analysis after first 30 days to identify competitor queries in the data."

**New account with no data**
Note it in context-pack.md. Set up conversion tracking before running any campaigns. Use `/google-ads-conversion-tracking` skill.

**Existing account with complex structure**
Don't try to document everything upfront. Fill what's known from the interview, then update context-pack.md after the first audit.

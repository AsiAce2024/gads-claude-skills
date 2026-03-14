# Google Ads AI Agent — Getting Started Guide

This guide walks you through setting up an AI agent that can read your live Google Ads data and apply deep optimization expertise to it. Once set up, you can ask natural language questions like *"What's driving my CPA spike this week?"* and get data-backed, expert answers — without pulling a single export.

---

## What you get

When this is set up, your AI agent (Claude, Cursor, or similar) will be able to:

**Query your live Google Ads data**
- Pull campaign performance, keywords, search terms, ad groups, and spend — across any date range
- Ask in plain English: no SQL, no dashboards, no exports

**Apply expert-level analysis**
- 21 specialized optimization skills that give the AI deep Google Ads domain knowledge
- Skills cover: account audits, negative keywords, search term mining, Quality Score, bidding, attribution, audience strategy, anomaly detection, and more

**Work across your full account structure**
- Supports individual accounts and MCC (manager) accounts
- Run the same analysis across multiple clients in one session (depending on connector)

**Example things you can do on day one:**
```
Run a full account audit and prioritize the top 5 things to fix.
```
```
Which search terms are spending money but have zero conversions in the last 30 days?
```
```
My CPA went up 40% this week — diagnose what happened.
```
```
Build me a negative keyword list from last month's search term report.
```
```
What's my Quality Score distribution and where should I focus first?
```

---

## How it works

The setup has three layers:

```
Your Google Ads Account
        ↓
  Data Connector (Windsor / Supermetrics / Official MCP)
        ↓
  AI Agent (Claude / Cursor) + Google Ads Skills
```

1. **Data connector** — bridges your live Google Ads data to the AI agent via MCP (Model Context Protocol), an open standard that lets AI tools query external data sources
2. **AI agent** — the tool you use to talk to the AI (Claude, Cursor, etc.)
3. **Skills** — a set of knowledge files installed into the agent that give it expert-level Google Ads frameworks on top of your live data

---

## Step 1 — Choose your data connector

Three options, ordered from easiest to most technical:

| Connector | Best for | Setup time | Cost |
|-----------|----------|------------|------|
| [**Windsor**](#option-a-windsor-easiest) | Anyone — fastest setup, works with Claude.ai directly | 10–15 min | Paid (free trial) |
| [**Supermetrics**](#option-b-supermetrics) | Already on Supermetrics, or want 200+ platforms in one workspace | 10–20 min | Existing plan |
| [**Official Google Ads MCP**](#option-c-official-google-ads-mcp-advanced) | Maximum data depth, no third-party cost, technical users | 30–60 min | Free |

**Not sure which to pick?** Start with Windsor. If you already use Supermetrics for reporting, use that instead. Only go with the Official MCP if you want full GAQL control or want to avoid third-party tools.

---

## Option A — Windsor (easiest)

Windsor connects your Google Ads account to your AI agent in about 10 minutes. No API tokens, no config files, no Google Cloud setup.

### A1. Create a Windsor account

1. Go to [onboard.windsor.ai](https://onboard.windsor.ai) and sign up.
2. Start the free 30-day trial — no credit card required.

> 📸 **Screenshot:** Windsor homepage at onboard.windsor.ai — capture the sign-up form or the "Start free trial" button

### A2. Connect Google Ads

1. Once inside Windsor, click **Add Data Source**.
2. Select **Google Ads** from the list.

> 📸 **Screenshot:** Windsor dashboard showing "Add Data Source" button and the platform selection screen with Google Ads highlighted

3. Click **Connect** and sign in with your Google Account.
4. Authorize read-only access when prompted.
5. Select which Google Ads accounts to include — you can select your MCC to cover all sub-accounts at once.

> 📸 **Screenshot:** The Google OAuth authorization screen, and then the Windsor screen showing your connected Google Ads accounts listed

### A3. Connect Windsor to your AI agent

**If you use Claude.ai (web app — recommended)**

1. Go to [claude.ai/customize/connectors](https://claude.ai/customize/connectors) or navigate to **Settings → Integrations**.
2. Search for **Windsor.ai** in the connector directory.
3. Click **Add** and authorize with your Windsor account.
4. Set the permission level to **Always allow** for faster responses.

> 📸 **Screenshot:** Claude.ai Settings → Integrations page showing the Windsor connector listed and the "Add" / authorize flow

**If you use Claude Desktop app**

Edit this file: `~/Library/Application Support/Claude/claude_desktop_config.json`

Add the following (replace with your Windsor API key):
```json
{
  "mcpServers": {
    "windsor": {
      "command": "mcp-proxy",
      "args": ["https://mcp.windsor.ai/", "--transport=streamablehttp"],
      "env": {
        "API_ACCESS_TOKEN": "YOUR_WINDSOR_API_KEY"
      }
    }
  }
}
```

First install `mcp-proxy`:
```bash
uv tool install mcp-proxy
```

**If you use Cursor**

Open **Settings → Tools & Integrations → New MCP Server** and add:
```json
{
  "mcpServers": {
    "windsor": {
      "url": "https://mcp.windsor.ai/"
    }
  }
}
```

A **"Needs Login"** button will appear — click it to authorize via browser.

> 📸 **Screenshot:** Cursor Settings showing the MCP server entry for Windsor with a green "Connected" status indicator

### A4. Verify Windsor is connected

In your agent, type:
```
What data sources do I have connected in Windsor?
```

You should see your Google Ads account(s) listed in the response.

> 📸 **Screenshot:** Claude or Cursor showing a response that lists your connected Windsor data sources, including Google Ads

---

## Option B — Supermetrics

If you're already using Supermetrics for reporting, this adds Google Ads live data to your AI agent in a few steps.

### B1. Connect Google Ads in Supermetrics Hub

1. Go to [hub.supermetrics.com](https://hub.supermetrics.com) and log in.
2. Navigate to **Data Sources** and click **Add connection**.
3. Select **Google Ads** and sign in with your Google Account.
4. Grant read-only access and select your ad accounts (MCC supported).

> 📸 **Screenshot:** Supermetrics Hub showing the Data Sources page with Google Ads listed as a connected source, showing account names

### B2. Get your Supermetrics API key

1. In the Supermetrics Hub, go to **Settings → API Keys** (or **Profile → API Keys**).
2. Click **Generate new key**.
3. Copy it immediately and store it somewhere safe (it won't be shown again).

> 📸 **Screenshot:** Supermetrics Hub API Keys page with the "Generate new key" button visible, and a blurred/redacted example of what a key looks like

### B3. Connect to your AI agent

**Claude.ai (web)**

1. Go to **Settings → Integrations** and search for **Supermetrics**.
2. Click Add and authenticate.

**Claude Desktop or Cursor**

```json
{
  "mcpServers": {
    "supermetrics": {
      "url": "https://mcp.supermetrics.com/mcp",
      "headers": {
        "Authorization": "Api-Key YOUR_SUPERMETRICS_API_KEY"
      }
    }
  }
}
```

> 📸 **Screenshot:** The final connected state — Claude or Cursor showing Supermetrics MCP as active, with an example query response showing Google Ads campaign data

### B4. Verify Supermetrics is connected

```
What data sources do I have connected in Supermetrics?
```

---

## Option C — Official Google Ads MCP (advanced)

Google's open-source MCP server gives full GAQL access to your Google Ads data. More setup required, but no third-party dependency.

**Prerequisites:** Python (via pipx), a Google Ads Developer Token, a Google Cloud project.

### C1. Install pipx

```bash
brew install pipx
pipx ensurepath
```

### C2. Get a Google Ads Developer Token

1. Go to your [Google Ads Manager account](https://ads.google.com).
2. Navigate to **Admin → API Center**.
3. Apply for API access — approval can take up to 48 hours.
4. Copy your Developer Token once approved.

> 📸 **Screenshot:** Google Ads Manager account showing the API Center screen (Admin → API Center) with the Developer Token visible (blurred in the screenshot)

### C3. Enable the Google Ads API in Google Cloud

1. Go to [console.cloud.google.com](https://console.cloud.google.com).
2. Create a new project or select an existing one.
3. Navigate to **APIs & Services → Library**, search for **Google Ads API**, and enable it.

> 📸 **Screenshot:** Google Cloud Console showing the Google Ads API in the library with the "Enabled" status indicator

### C4. Configure OAuth credentials

In **APIs & Services → Credentials**, create an OAuth 2.0 Client ID (type: Desktop App). Download the JSON file.

Then run:
```bash
gcloud auth application-default login \
  --scopes https://www.googleapis.com/auth/adwords,https://www.googleapis.com/auth/cloud-platform \
  --client-id-file=YOUR_CLIENT_JSON_FILE
```

### C5. Add to your agent config

```json
{
  "mcpServers": {
    "google-ads-mcp": {
      "command": "pipx",
      "args": [
        "run", "--spec",
        "git+https://github.com/googleads/google-ads-mcp.git",
        "google-ads-mcp"
      ],
      "env": {
        "GOOGLE_APPLICATION_CREDENTIALS": "PATH_TO_CREDENTIALS_JSON",
        "GOOGLE_PROJECT_ID": "YOUR_PROJECT_ID",
        "GOOGLE_ADS_DEVELOPER_TOKEN": "YOUR_DEVELOPER_TOKEN"
      }
    }
  }
}
```

> 📸 **Screenshot:** Claude Desktop or Cursor showing the MCP config file open with the google-ads-mcp block, and then the agent responding to `/mcp` with the server listed

---

## Step 2 — Install the Google Ads Skills

Once your data connector is working, install the skills. This gives your agent deep Google Ads expertise — not just data access, but the frameworks and playbooks to do something useful with it.

Run this in your terminal:

```bash
npx skills add eliasmalm/google-ads-skills
```

> 📸 **Screenshot:** Terminal showing the `npx skills add` command running and completing successfully, with a list of skills installed

This installs 21 skills covering:

| Area | Skills included |
|------|----------------|
| Account health | Account audit, anomaly detection |
| Keywords | Keywords, negative keywords, search term mining, cannibalization |
| Ads | Ad copy, ad extensions, ad extension audit, Quality Score |
| Bidding & budget | Bidding, budget management |
| Measurement | Conversion tracking, attribution, UTM generator |
| Audiences | Audiences, segmentation |
| Testing | Experiments |
| Automation | Scripts |
| Audits | Ecommerce audit, lead gen audit |

---

## Step 3 — Test that everything works

Start with a simple query to confirm the data connection:

```
What are my top 10 campaigns by spend in the last 30 days?
```

Then try something that uses a skill:

```
Run a quick account audit — what are the top 3 things I should fix first?
```

```
Which search terms are spending budget but have zero conversions this month?
```

```
I have a CPA target of $50 — which campaigns are over target and why?
```

> 📸 **Screenshot:** Claude showing a detailed response to the account audit prompt — capture a response that includes structured findings, not just raw data (ideally showing the skill's framework being applied, e.g. a prioritized list of issues with financial impact)

---

## Troubleshooting

### Connector not showing up in Claude

- Restart the agent after any config file change
- Check the config file for JSON syntax errors (missing commas, unclosed brackets)
- On Claude Desktop, go to **Help → Restart Claude** to force a reload

### "Needs Login" not clearing (Windsor / Cursor)

- Refresh the MCP server in Cursor settings after completing OAuth in the browser
- Try disconnecting and reconnecting the server from the settings panel

### No data returned

- Confirm your Google Ads account is connected and has data in the date range you specified
- Check that the right accounts are selected in your connector dashboard (Windsor Hub or Supermetrics Hub)
- Try a broader date range — some accounts have delayed data

### Authentication errors (Supermetrics)

- The API key must be prefixed: `Authorization: Api-Key your-key` — not just the key alone
- Regenerate the key in hub.supermetrics.com if in doubt

### Developer token issues (Official MCP)

- Token approval can take up to 48 hours from Google
- Ensure you're applying from a **manager (MCC) account**, not a standard account
- Check the token status at **Admin → API Center** in Google Ads

### Skills not loading

- Run `npx skills add eliasmalm/google-ads-skills` again to reinstall
- Check you're running a recent version of Node.js (`node -v` should return v18+)

---

## Limitations

Understanding what this setup **cannot** do is as important as knowing what it can.

| Limitation | Detail |
|------------|--------|
| **Read-only** | None of the data connectors support writing back to Google Ads — the agent cannot create, pause, or modify campaigns, ads, or bids |
| **No real-time alerts** | The agent responds to questions — it doesn't proactively monitor your account. You have to initiate a session to get insights |
| **Historical data range** | Depends on your connector. Windsor and Supermetrics typically provide up to 36 months of history. The Official MCP depends on what's in your Google Ads account |
| **Data freshness** | Google Ads data typically has a 3-hour lag. Same-day data may be incomplete. For Windsor and Supermetrics, check their connector sync frequency |
| **Cost data vs. conversion data** | Conversion data (especially offline conversions and attribution model differences) may not always match what you see in the Google Ads UI — this is a Google data discrepancy, not a connector issue |
| **PMax opacity** | Performance Max campaign data is inherently limited even via the API — search term reports, asset-level breakdowns, and channel splits are not fully accessible |
| **Rate limits** | The Official Google Ads MCP is subject to Google Ads API rate limits. For large accounts with many queries in a session, you may hit limits |
| **Multi-account MCC queries** | Querying across all sub-accounts simultaneously is possible but can be slow for large MCCs — it's better to query account by account |

---

## Resources

| Resource | Link |
|----------|------|
| GitHub — All skills and setup guides | [github.com/eliasmalm/google-ads-skills](https://github.com/eliasmalm/google-ads-skills) |
| Detailed Windsor setup | [setup/windsor-mcp.md](setup/windsor-mcp.md) |
| Detailed Supermetrics setup | [setup/supermetrics-mcp.md](setup/supermetrics-mcp.md) |
| Detailed Official MCP setup | [setup/google-ads-mcp.md](setup/google-ads-mcp.md) |
| Windsor.ai | [windsor.ai](https://windsor.ai) |
| Supermetrics Hub | [hub.supermetrics.com](https://hub.supermetrics.com) |
| Official Google Ads MCP (GitHub) | [github.com/googleads/google-ads-mcp](https://github.com/googleads/google-ads-mcp) |
| Google Ads Developer Token | [Google Ads API Center](https://ads.google.com) — Admin → API Center |

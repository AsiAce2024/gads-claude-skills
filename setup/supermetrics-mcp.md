# Setup: Supermetrics MCP

[Supermetrics](https://supermetrics.com) is a marketing data platform that connects 200+ data sources — including Google Ads — to your AI agent through a single MCP server. If you already use Supermetrics for reporting, this is the fastest path to live Google Ads data in your agent.

> **Best for:** Teams already on Supermetrics, or who want access to Google Ads alongside other platforms (Meta, LinkedIn, TikTok, GA4, etc.) in the same session.

---

## What you can do with it

- "What was my Google Ads spend last week by campaign?"
- "Compare Google Ads and Meta ROAS for Q1."
- "Which ad groups have a CTR below 1% with over 500 impressions this month?"
- "Show me conversion rate by device for the last 30 days."

Supermetrics MCP gives access to Google Ads metrics, dimensions, and filtering — directly queryable by your agent through natural language.

---

## Prerequisites

- A [Supermetrics account](https://supermetrics.com) — plans vary; MCP access depends on your subscription tier
- Google Ads connected in your Supermetrics Hub
- Your Supermetrics API key
- Your AI agent installed: Claude Desktop, Claude.ai (Pro/Team), Cursor, or another MCP-compatible tool

---

## Step 1 — Connect Google Ads to Supermetrics

1. Go to [hub.supermetrics.com](https://hub.supermetrics.com) and sign in.
2. Navigate to **Data Sources** and find **Google Ads**.
3. Click **Add connection** and sign in with your Google Account.
4. Grant the required access scopes (read-only is fine).
5. Select which ad accounts to include — you can connect a manager (MCC) account to cover all sub-accounts.

---

## Step 2 — Get your Supermetrics API key

1. In [hub.supermetrics.com](https://hub.supermetrics.com), go to **Settings → API Keys** (or **Profile → API Keys**).
2. Click **Generate new key** and copy it immediately — it won't be shown again.
3. Store it somewhere safe (e.g. a password manager or `.env` file).

> **Keep this key private.** Anyone with it can query all data sources connected to your Supermetrics account.

---

## Step 3 — Connect Supermetrics MCP to your agent

Choose your agent below.

### Claude.ai (web — Pro or Team plan)

1. Go to [claude.ai/directory](https://claude.ai/directory) or **Settings → Integrations**.
2. Search for **Supermetrics** and click to add it.
3. Authenticate with your Supermetrics account when prompted.
4. Set permissions to **Always allow** for faster access during sessions.

This is the easiest method — no config files, no CLI.

---

### Claude Desktop (app)

Edit: `~/Library/Application Support/Claude/claude_desktop_config.json`

```json
{
  "mcpServers": {
    "supermetrics": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://mcp.supermetrics.com/mcp",
        "--header",
        "Authorization: Api-Key YOUR_SUPERMETRICS_API_KEY"
      ]
    }
  }
}
```

Replace `YOUR_SUPERMETRICS_API_KEY` with the key you generated in Step 2. Restart Claude Desktop after saving.

Alternatively, if you have `mcp-remote` installed globally:

```bash
npm install -g mcp-remote
```

---

### Cursor

**Method 1 — UI (recommended)**

1. Open **Cursor Settings → Tools & Integrations → New MCP Server**.
2. Fill in:
   - **Name**: `supermetrics`
   - **Type**: `streamableHttp`
   - **URL**: `https://mcp.supermetrics.com/mcp`
   - **Header**: `Authorization: Api-Key YOUR_SUPERMETRICS_API_KEY`
3. Click **Save** and restart Cursor.

**Method 2 — JSON config**

Add to `.cursor/mcp.json` in your project root (or global `~/.cursor/mcp.json`):

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

---

### Gemini CLI / Windsurf / Other MCP Clients

Add to your MCP config:

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

---

## Step 4 — Verify the connection

In your agent, ask:

```
What data sources do I have connected in Supermetrics?
```

Then try a Google Ads query:

```
What were my top 5 Google Ads campaigns by spend last week?
```

---

## Available Supermetrics MCP tools

| Tool | What it does |
|------|-------------|
| `data_source_discovery` | Lists all connected data sources and their metadata |
| `accounts_discovery` | Returns ad accounts available for a given data source |
| `field_discovery` | Returns available metrics and dimensions for a source |
| `data_query` | Retrieves structured data with filters, date ranges, and grouping |
| `get_async_query_results` | Retrieves results for longer-running queries |
| `get_today` | Returns today's date (used for relative date calculations) |

---

## Example prompts for Google Ads

```
Show me Google Ads performance by campaign for the last 30 days — spend, impressions, clicks, CPC, conversions, CPA, and ROAS.
```

```
Which Google Ads ad groups have a CTR below 2% and over 1,000 impressions this month?
```

```
Compare Google Ads and Meta Ads ROAS month-over-month for the last 3 months.
```

```
What's the conversion rate by device type for my Google Ads campaigns this quarter?
```

```
Which Google Ads keywords have the highest CPA in the last 14 days?
```

---

## Comparison: Supermetrics vs Windsor vs Official Google Ads MCP

| | Supermetrics MCP | Windsor MCP | Official Google Ads MCP |
|---|---|---|---|
| **Setup time** | 10–20 min | 10–15 min | 30–60 min |
| **Auth method** | API key | OAuth via Windsor | Google Cloud OAuth + Developer Token |
| **Data sources** | 200+ platforms | 325+ platforms | Google Ads only |
| **Google Ads depth** | Full metrics + dimensions | 430+ metrics, 1,905+ dimensions | Full GAQL access |
| **Write access** | Read-only | Read-only | Read-only |
| **Cost** | Paid (existing Supermetrics plan) | Paid (free trial available) | Free (open source) |
| **Best for** | Existing Supermetrics users, multi-channel | Quick multi-channel setup | Full GAQL control, no cost |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `401 Unauthorized` | Check the API key — ensure it starts with `Api-Key ` in the header |
| No data returned | Verify Google Ads is connected and active in hub.supermetrics.com |
| Wrong account data | Check which accounts are selected in your Supermetrics data source settings |
| Tools not appearing | Restart the agent after adding the MCP config |
| Query timeout | Use `get_async_query_results` for large date ranges or many dimensions |

---

## Next step

Once connected, [install the Google Ads Skills](../README.md#install) to give your agent deep domain knowledge on top of live data access.

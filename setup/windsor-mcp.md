# Setup: Windsor MCP (Third-Party, No-Code)

[Windsor](https://windsor.ai) is a no-code data connector that gives your AI agent access to Google Ads — and 325+ other marketing platforms — through a single MCP server. It takes about 10–15 minutes to set up and requires no API tokens, OAuth flows, or Google Cloud configuration.

> **Best for:** Marketers who want quick setup, or who need to query multiple ad platforms (Meta, TikTok, LinkedIn, etc.) alongside Google Ads in the same agent session.

---

## What you can do with it

- "What was my Google Ads spend last week broken down by campaign?"
- "Compare ROAS across Google Ads, Meta, and TikTok for Q1."
- "Which Google Ads keywords have the highest CPA this month?"
- "Show me impression share by campaign for the last 30 days."

Windsor provides 430+ Google Ads metrics and 1,905+ dimensions.

---

## Prerequisites

- A Windsor.ai account — [start a free 30-day trial](https://onboard.windsor.ai/)
- A Google Ads account (individual or MCC)
- Your AI agent installed: Claude Desktop, Cursor, or other MCP-compatible tool

---

## Step 1 — Connect Google Ads to Windsor

1. Sign up or log in at [onboard.windsor.ai](https://onboard.windsor.ai/).
2. Click **Add Data Source** and select **Google Ads**.
3. Authorize read-only access via OAuth — this takes about 1 minute.
4. Select which ad accounts to include (individual accounts or your full MCC).

---

## Step 2 — Connect Windsor MCP to your agent

Choose your agent below.

### Claude.ai (web — easiest, recommended)

1. Go to [claude.ai/customize/connectors](https://claude.ai/customize/connectors) or **Settings → Integrations**.
2. Search for **Windsor.ai** and click **+** to add it.
3. Authorize with your Windsor account via OAuth — takes about 1 minute.
4. Set permissions to **Always allow** for faster access during sessions.

No config files, no CLI, no API keys. This is the path most people should use.

---

### Claude Desktop (app — developer config)

Edit: `~/Library/Application Support/Claude/claude_desktop_config.json`

```json
{
  "mcpServers": {
    "windsor": {
      "command": "mcp-proxy",
      "args": [
        "https://mcp.windsor.ai/",
        "--transport=streamablehttp"
      ],
      "env": {
        "API_ACCESS_TOKEN": "YOUR_WINDSOR_API_KEY"
      }
    }
  }
}
```

To install `mcp-proxy`:

```bash
uv tool install mcp-proxy
```

Find the installed path with:

```bash
which mcp-proxy
```

Replace `"command": "mcp-proxy"` with the full path if needed.

Restart Claude Desktop after editing the config.

### Cursor

1. Open **Settings → Tools & Integrations → New MCP Server**.
2. This opens `mcp.json`. Add the following:

```json
{
  "mcpServers": {
    "windsor": {
      "url": "https://mcp.windsor.ai/"
    }
  }
}
```

3. Save the file. A **"Needs Login"** button will appear — click it.
4. This opens a browser window for Windsor OAuth. Authorize and return to Cursor.
5. The server status will update to connected.

### Gemini CLI / Windsurf / Other MCP Clients

Add to your MCP config file:

```json
{
  "mcpServers": {
    "windsor": {
      "url": "https://mcp.windsor.ai/"
    }
  }
}
```

Then authenticate via the browser prompt when the agent first attempts to connect.

---

## Step 3 — Verify the connection

In your agent, ask:

```
What data sources do I have connected in Windsor?
```

Then try a Google Ads query:

```
What were my top 5 campaigns by spend last week in Google Ads?
```

---

## Available Windsor MCP functions

| Function | What it does |
|----------|-------------|
| `get_connectors` | Lists your connected data sources and ad accounts |
| `get_options` | Returns available metrics, dimensions, and date filter options |
| `get_fields` | Returns detailed metadata about specific fields |
| `get_data` | Retrieves structured marketing data filtered by fields, dates, and segments |

---

## Example prompts for Google Ads

```
Show me Google Ads performance by campaign for the last 30 days — include spend, clicks, conversions, and ROAS.
```

```
Which Google Ads ad groups have a CTR below 2% and more than 1,000 impressions this month?
```

```
Compare Google Ads spend month-over-month for the last 3 months by campaign type.
```

```
What's the impression share trend for my brand campaigns over the last 60 days?
```

---

## Comparison: Windsor vs Official Google Ads MCP

| | Windsor MCP | Official Google Ads MCP |
|---|---|---|
| **Setup time** | 10–15 min | 30–60 min |
| **Auth method** | OAuth via Windsor (no tokens) | Google Cloud OAuth + Developer Token |
| **Data sources** | 325+ (Google Ads, Meta, TikTok, etc.) | Google Ads only |
| **Google Ads metrics** | 430+ metrics, 1,905+ dimensions | Full GAQL access |
| **Write access** | Read-only | Read-only |
| **Cost** | Paid (free trial available) | Free (open source) |
| **Best for** | Multi-channel, quick setup | Full GAQL control, no cost |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Needs Login" not clearing | Refresh the MCP server in settings after authorizing in browser |
| No data returned | Verify at least one data source is connected in Windsor dashboard |
| Wrong account data | Check which accounts are selected in Windsor's connector settings |
| `mcp-proxy` not found | Run `uv tool install mcp-proxy` and use `which mcp-proxy` to find full path |

---

## Next step

Once connected, [install the Google Ads Skills](../README.md#install) to give your agent deep domain knowledge on top of live data access.

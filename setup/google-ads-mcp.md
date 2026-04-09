# Setup: Official Google Ads MCP (Model Context Protocol)

Google officially open-sourced the [Google Ads API MCP Server](https://github.com/googleads/google-ads-mcp) in October 2025. It connects your AI agent (Claude, Cursor, Gemini CLI, etc.) directly to your Google Ads account via natural language — no manual data exports required.

> **Note:** The official MCP server is currently **read-only** — reporting and diagnostics only. It does not support creating or modifying campaigns.

---

## What you can do with it

- "How is my campaign performance this week?"
- "What search terms drove the most clicks last month?"
- "Which campaigns are over budget this month?"
- "How many active campaigns do I have?"

---

## Prerequisites

- Python installed (via [pipx](https://pipx.pypa.io/stable/#install-pipx))
- A Google Ads account (individual or manager/MCC account)
- A Google Cloud project with billing enabled
- A Google Ads Developer Token

---

## Step 1 — Install pipx

```bash
brew install pipx
pipx ensurepath
```

Or follow the [official pipx install guide](https://pipx.pypa.io/stable/#install-pipx).

---

## Step 2 — Get a Google Ads Developer Token

1. Log in to your [Google Ads manager (MCC) account](https://ads.google.com).
2. Go to **Admin → API Center**.
3. Apply for API access — fill in the form with your intended use.
4. Wait for approval (can take up to 48 hours).
5. Once approved, copy your **Developer Token**.

> If you don't have a manager account, [create one for free](https://ads.google.com/home/tools/manager-accounts/) — even if you only manage a single account.

---

## Step 3 — Enable the Google Ads API in Google Cloud

1. Go to [Google Cloud Console](https://console.cloud.google.com/).
2. Create a new project (or select an existing one).
3. Navigate to **APIs & Services → Library**.
4. Search for **Google Ads API** and enable it.

---

## Step 4 — Configure OAuth Credentials

### Option A — Application Default Credentials (recommended)

Download your OAuth client JSON from **APIs & Services → Credentials → Create OAuth Client ID** (type: Desktop App).

Then run:

```bash
gcloud auth application-default login \
  --scopes https://www.googleapis.com/auth/adwords,https://www.googleapis.com/auth/cloud-platform \
  --client-id-file=YOUR_CLIENT_JSON_FILE
```

When complete, note the `PATH_TO_CREDENTIALS_JSON` printed to the console — you'll need it in the next step.

### Option B — Google Ads Python client library (`google-ads.yaml`)

Follow the [Python client library setup guide](https://developers.google.com/google-ads/api/docs/client-libs/python/). If you already have a working `google-ads.yaml`, you can reuse it.

In `utils.py`, change `get_googleads_client()` to use the `load_from_storage()` method.

---

## Step 5 — Configure your AI agent

Add the MCP server to your agent's config file. Choose your agent below.

### Claude Desktop

Edit: `~/Library/Application Support/Claude/claude_desktop_config.json`

```json
{
  "mcpServers": {
    "google-ads-mcp": {
      "command": "pipx",
      "args": [
        "run",
        "--spec",
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

### Cursor

1. Open **Settings → Tools & Integrations → New MCP Server**.
2. This opens `mcp.json`. Add the following:

```json
{
  "mcpServers": {
    "google-ads-mcp": {
      "command": "pipx",
      "args": [
        "run",
        "--spec",
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

### Gemini CLI

Edit: `~/.gemini/settings.json`

```json
{
  "mcpServers": {
    "google-ads-mcp": {
      "command": "pipx",
      "args": [
        "run",
        "--spec",
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

### Manager (MCC) Account

If your access to the ad account is through a manager account, add the manager customer ID to the `env` block:

```json
"GOOGLE_ADS_LOGIN_CUSTOMER_ID": "YOUR_MANAGER_CUSTOMER_ID"
```

---

## Step 5b — CRITICAL: Apply the stdio-hang patch (Windows + any stdio MCP host)

> **You must do this or every tool call will hang forever.**

The public release of `google-ads-mcp` (0.0.1) calls `google.auth.default()` inside every tool invocation. In an MCP stdio subprocess context, this call **deadlocks indefinitely** — no error, no timeout, the Python process sits at 0% CPU forever. Symptoms: the server starts, `tools/list` returns instantly, but the moment you ask Claude/Cursor/Gemini a real question, nothing happens.

The fix is a small patch to `ads_mcp/utils.py` that bypasses `google.auth.default()` and reads the ADC JSON directly. The patch script is idempotent and creates a backup.

```bash
# Adjust paths if your venv is elsewhere.
C:\Users\%USERNAME%\.venvs\google-ads-mcp\Scripts\python.exe \
  C:\Users\%USERNAME%\.claude\scripts\google-ads-mcp\apply-stdio-hang-patch.py \
  C:\Users\%USERNAME%\.venvs\google-ads-mcp
```

Then verify:

```bash
set GOOGLE_ADS_DEVELOPER_TOKEN=YOUR_DEV_TOKEN
set GOOGLE_ADS_LOGIN_CUSTOMER_ID=YOUR_MCC_ID_NO_DASHES
set PYTHONUTF8=1
python C:\Users\%USERNAME%\.claude\scripts\google-ads-mcp\verify-mcp-health.py
```

Expected: `HEALTHY` with cold call <30s and warm call <5s.

> **Re-run the patch after every reinstall or upgrade.** `pip install --upgrade google-ads-mcp` will overwrite the patched file.

---

## Step 6 — Verify the connection

In your agent, type `/mcp` — you should see `google-ads-mcp` listed.

Then try:

```
What customers do I have access to?
```

```
How many active campaigns do I have for customer id 1234567890?
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| **Tool calls hang forever** | The stdio-hang patch (Step 5b) was not applied — or `pip` overwrote it on a recent reinstall. Re-run the patch script. |
| Permission denied | Make sure you're using your MCC customer ID in `GOOGLE_ADS_LOGIN_CUSTOMER_ID` |
| Empty results | Check that your date range is correct and the account has spend |
| Token not approved | Developer token requests can take up to 48 hours |
| `gcloud` not found | Install [Google Cloud CLI](https://cloud.google.com/sdk/docs/install) |

---

## Available tools (read-only)

| Tool | What it does |
|------|-------------|
| `search` | Executes GAQL queries — fetch campaigns, ad groups, keywords, metrics |
| `list_accessible_customers` | Lists all accounts you have access to |

---

## Next step

Once connected, [install the Google Ads Skills](../README.md#install) to give your agent deep domain knowledge on top of live data access.

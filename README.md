# Google Ads Skills

**21 Google Ads optimization skills for AI agents.**

Connect your live Google Ads data to Claude, Cursor, or any MCP-compatible agent — then install skills that give it deep optimization expertise. Ask natural language questions, get expert-level analysis back.

```
"Run a full account audit and tell me the top 5 things to fix."
"Which search terms are spending money with zero conversions this month?"
"My CPA went up 40% this week — what happened?"
"Build a negative keyword list from last month's search term report."
```

---

## Quickstart

**Step 1 — Connect your Google Ads account** (pick one):

| Connector | Best for | Time |
|-----------|----------|------|
| [Windsor MCP](setup/windsor-mcp.md) | Fastest setup, no API tokens, 325+ platforms | 10–15 min |
| [Supermetrics MCP](setup/supermetrics-mcp.md) | Already on Supermetrics, 200+ platforms | 10–20 min |
| [Official Google Ads MCP](setup/google-ads-mcp.md) | Full GAQL control, no third-party cost | 30–60 min |

New here? The **[Getting Started Guide](getting-started.md)** walks through each option with screenshots and troubleshooting.

**Step 2 — Install the skills:**

```bash
npx skills add eliasmalm/google-ads-skills
```

---

## Skills

All 21 skills are task and optimization focused — deep playbooks for doing specific jobs, not generic campaign overviews.

| Skill | What it does |
|-------|-------------|
| [`google-ads-account-audit`](skills/google-ads-account-audit/SKILL.md) | Full account audit, prioritize fixes by financial impact, first-30-days takeover plan |
| [`google-ads-keywords`](skills/google-ads-keywords/SKILL.md) | Keyword research, match type strategy, intent classification, keyword architecture |
| [`google-ads-negative-keywords`](skills/google-ads-negative-keywords/SKILL.md) | Eliminate wasted spend, build negative lists, prevent campaign cannibalization |
| [`google-ads-search-term-mining`](skills/google-ads-search-term-mining/SKILL.md) | Mine search term reports for keyword opportunities, intent signals, and structural insights |
| [`google-ads-ad-copy`](skills/google-ads-ad-copy/SKILL.md) | Write RSA headlines and descriptions, intent-matched frameworks for every ad group type |
| [`google-ads-ad-extensions`](skills/google-ads-ad-extensions/SKILL.md) | Extension strategy, write sitelinks and callouts, optimize for CTR and Ad Rank |
| [`google-ads-ad-extension-audit`](skills/google-ads-ad-extension-audit/SKILL.md) | Coverage matrix across campaigns, sitelink/callout quality checks, prioritized fix list |
| [`google-ads-quality-score`](skills/google-ads-quality-score/SKILL.md) | Diagnose QS by component, fix Expected CTR / Ad Relevance / LPE, reduce CPCs |
| [`google-ads-bidding`](skills/google-ads-bidding/SKILL.md) | Smart Bidding strategies, tCPA, tROAS, when to use each, portfolio bid strategies |
| [`google-ads-budget-management`](skills/google-ads-budget-management/SKILL.md) | Budget pacing, allocation, fixing underspend/overspend, Smart Bidding × budget |
| [`google-ads-audiences`](skills/google-ads-audiences/SKILL.md) | RLSA, Customer Match, in-market, lookalikes, audience exclusions |
| [`google-ads-segmentation`](skills/google-ads-segmentation/SKILL.md) | Device, geo, and dayparting analysis — bid adjustments, campaign splits |
| [`google-ads-attribution`](skills/google-ads-attribution/SKILL.md) | Attribution models, data-driven attribution, conversion windows, impact on Smart Bidding |
| [`google-ads-conversion-tracking`](skills/google-ads-conversion-tracking/SKILL.md) | Google Tag, GTM setup, enhanced conversions, offline conversion import |
| [`google-ads-experiments`](skills/google-ads-experiments/SKILL.md) | Campaign experiments, A/B tests, statistical significance, bid strategy tests |
| [`google-ads-anomaly-detection`](skills/google-ads-anomaly-detection/SKILL.md) | Diagnose performance drops and spikes, proactive monitoring setup |
| [`google-ads-scripts`](skills/google-ads-scripts/SKILL.md) | JavaScript automation — budget monitoring, keyword pausing, QS tracking, reporting |
| [`google-ads-keyword-cannibalization`](skills/google-ads-keyword-cannibalization/SKILL.md) | Find campaigns competing against each other, calculate cost, implement query routing |
| [`google-ads-audit-ecommerce`](skills/google-ads-audit-ecommerce/SKILL.md) | ROAS audit, product feed health, Shopping/PMax structure, cart abandonment, seasonality |
| [`google-ads-audit-leadgen`](skills/google-ads-audit-leadgen/SKILL.md) | CPL audit, lead quality signals, offline conversion import, form optimization, B2B targeting |
| [`google-ads-utm-generator`](skills/google-ads-utm-generator/SKILL.md) | Build UTM templates, dynamic value insertion, naming conventions, GA4 channel grouping |

Skills are interconnected — `google-ads-search-term-mining` feeds `google-ads-negative-keywords`, `google-ads-conversion-tracking` is a prerequisite for `google-ads-bidding`, and `google-ads-attribution` directly affects how Smart Bidding performs.

---

## How it works

```
Google Ads Account
      ↓
Data Connector  ←  Windsor / Supermetrics / Official Google Ads MCP
      ↓
AI Agent        ←  Claude / Cursor / Windsurf / Gemini CLI
      ↓
Skills          ←  npx skills add eliasmalm/google-ads-skills
```

The data connector gives the agent access to your live account data. The skills give it the domain expertise to do something useful with it.

---

## Compatible with

Claude · Cursor · Windsurf · Gemini CLI · Codex · Any MCP-compatible agent

---

## License

MIT

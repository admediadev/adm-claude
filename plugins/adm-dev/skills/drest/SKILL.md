---
name: drest
description: "Developer - clone production database tables into the dev database using the internal drest command. Use for drest, refresh dev data, copy prod table to dev, clone table to dev."
---

Clone database tables from PROD to DEV using the internal command `drest`.

Follow this workflow:

## 1. Detect Tables

Scan the conversation and list all referenced DB tables.
Group them by database and briefly note why each table was used.

Common databases:
- **Admin** — advertiser info, budgets, portal users, announcements, daily_budget_stats
- **Shorty** — campaign, conversion tracking
- **keywords** — partitioned click tables (adv_clicks_*, adv_fb_*, adv_aggr_*), keyword_daily_stats
- **whale** — whale-specific data

**Important:** The MySQL database name is `keywords` (not `keyword`). Use `drest keywords ...`.

## 2. Ask WHERE Clause (for large tables)

For large or daily-partitioned tables, ask the user to choose a filtering condition:

- `date='2026-04-08'`
- `date BETWEEN '2026-04-01' AND '2026-04-08'`
- `date='2026-04-08' AND adv_id=12345`
- `date='2026-04-08' AND campaign_id=XXXXX`
- Custom condition
- No WHERE clause

Skip the WHERE question for small reference tables (e.g., Advertiser_info, campaign, portal_users).
If uncertain whether a table is large, ask.

## 3. Generate Commands

Format: `drest [db] [table] "[WHERE clause]"`

Examples:
```
drest Admin Advertiser_info
drest Admin aff_cpc_summary "date='2026-04-08' and adv_id=12345"
drest Shorty campaign
drest keywords keyword_daily_stats "date='2026-04-08'"
```

## 4. Confirmation

Present the full list of commands and ask the user to confirm which ones to run.

## 5. Execute

Run the confirmed commands sequentially and report results.

## Goal

Clone all tables referenced in the conversation safely using `drest`, with WHERE clauses for large tables and user confirmation before execution.

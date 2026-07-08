---
name: dev-mysql
description: Query the local dev MySQL server (which mirrors prod via drest). Use when verifying DB-side behavior of code changes — partition contents, daily_budget_stats rollups, schema, ad-hoc spot checks.
---

# dev-mysql

The dev MySQL server runs on `127.0.0.1` on this box, with all the AdMedia DBs
co-located: `Admin`, `keywords`, `shorty`, `whale`, `stats`, `arch`, `search`,
`cms`, `adcenter`, `the_agency`, etc. In prod these are on separate hosts; the
`drest` command (`/usr/bin/drest`) syncs prod tables into these dev DBs so we
can test changes against realistic data.

## Tool

`bin/devsql` (in this repo). Wrapper that resolves credentials from the
matching `/usr/share/php/*_oci.ini` and runs `mysql`.

```
bin/devsql <db> "<sql>"            # read-only (SELECT/SHOW/DESC/EXPLAIN/USE/SET/WITH)
bin/devsql --write <db> "<sql>"    # allow INSERT/UPDATE/DELETE/DDL
bin/devsql --tsv <db> "<sql>"      # tab-separated output (default is table)
bin/devsql <db> < file.sql         # SQL from stdin
bin/devsql --list                  # show DB → ini file map
```

If invoked from elsewhere, use the absolute path
`/var/www/php74/mason/bin/devsql`.

## Supported DBs

`Admin`, `keywords`, `shorty`, `whale`, `stats`, `arch`, `search`, `cms`,
`adcenter`, `the_agency`, `rtb`, `live`, `montanapost`, `local_pages`,
`feeds`, `email_whois`, `amazon`. Run `devsql --list` for the live map.

## When to use

- Verifying a controller/service/cron change actually returns the expected rows
- Inspecting schema (`DESC table`, `SHOW CREATE TABLE table`)
- Spot-checking a partition table (`adv_clicks_<adv_id>_<YYYYMM>` in `keywords`)
- Confirming `daily_budget_stats` rollups (in `Admin`) match what the dashboard renders
- Counting rows after a `drest` to confirm the sync succeeded

## When NOT to use

- This only points at the dev server. Never use it as a stand-in for "does prod
  look right?"
- For schema migrations the user wants as `.sql` files for manual run, **produce
  the `.sql` file** — do not apply it yourself with `--write`.

## Conventions

- **Read-only by default.** `--write` is required for any non-SELECT and you
  should state in chat what you're about to write before running it.
- **Always filter partition tables by date** (`adv_clicks_*`, `adv_fb_*`, etc.)
  — they can be huge even on dev.
- For multi-line or complex SQL, write to a temp file and pipe via stdin
  rather than embedding in `-e`.
- Default output is the standard MySQL table format. Use `--tsv` when you need
  to count, sort, or otherwise post-process rows.
- Credentials live in `/usr/share/php/*_oci.ini` — that's the source of truth
  if connection ever fails.

## Examples

```bash
# Schema check
bin/devsql Admin "DESC daily_budget_stats"

# Row count of a partition
bin/devsql keywords "SELECT COUNT(*) FROM adv_clicks_12345_202604 WHERE date = '2026-04-15'"

# Compare a dashboard KPI against the rollup
bin/devsql Admin "SELECT date, SUM(clicks), SUM(spent) FROM daily_budget_stats
                  WHERE adv_id = 12345 AND date BETWEEN '2026-04-01' AND '2026-04-15'
                  GROUP BY date ORDER BY date"

# Apply a one-off test fixture (write mode)
bin/devsql --write Admin "UPDATE Advertiser_info SET status = 'paused' WHERE adv_id = 99999"
```

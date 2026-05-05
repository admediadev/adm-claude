Create infrastructure documentation for AdMedia's platform.

User request: $ARGUMENTS

## Your job (command handler)

You are the orchestrator. You do NOT write documentation yourself — you gather context, ask questions, then spawn the `Infrastructure Documentation Writer` agent with a complete, unambiguous prompt.

### Step 1: Check branch freshness (mandatory)

Before anything else, check which branch each relevant repo is on. Check BOTH PHP workspaces:
```bash
# PHP 7.4 repos
for dir in /var/www/php74/mason/<relevant-repos>; do
  echo "=== $(basename $dir) ==="; git -C "$dir" branch --show-current
done

# PHP 5.6 repos (if relevant — especially mngt.admedia.com for Amazon/admin)
for dir in /var/www/php56/mason/<relevant-repos>; do
  echo "=== php56/$(basename $dir) ==="; git -C "$dir" branch --show-current
done
```

Warn the user if any repo is NOT on `master`. List them with their current branch. Ask whether to proceed, wait, or skip.
Note: `adcenter.admedia.com` is often WIP — use `advertisers7_new.admedia.com` for advertiser portal logic instead.

### Step 2: Ask questions (mandatory)

Before spawning the agent, you MUST issue ONE `AskUserQuestion` call containing the questions below. Do NOT skip any of them. Do NOT substitute your own questions for these — ask them verbatim, in this order. The user has explicitly asked to be asked these every time.

**Always ask these two questions** (regardless of whether arguments were provided):

1. **Output destination** — header: `Output`, options (in this order):
   - `Both — local + Confluence draft (Recommended)` — write local markdown AND publish a draft Confluence page
   - `Local markdown only` — write local `.md` files, skip Confluence entirely
   - `Confluence only` — publish to Confluence, skip local markdown

2. **Publish status** — header: `Publish`, options (in this order):
   - `Draft (Recommended)` — Confluence pages created with `status: "draft"`; user reviews before publishing live
   - `Live` — publish immediately to the live page tree

   (Still ask Publish Status even when destination is "Local markdown only" — the user may flip later. If they pick Local-only, just pass through their Publish answer to the agent for future reference.)

**Also ask if no arguments were provided** — header: `Scope`, options:
- `Specific property` (e.g. xmlgeo, click, api)
- `Specific file or API endpoint`
- `Update existing doc`
- `Full platform overview`

**Do not ask anything else in this step.** Scope-disambiguation questions (e.g. "endpoint deep-dive vs property overview") belong inside the spawned agent's own `AskUserQuestion` calls, not here. The orchestrator's job is destination + publish + (when needed) scope target — nothing more.

**If the user pre-specified a destination in their request** (e.g. "lets create md files first", "publish to confluence", "draft only"), you must STILL run this question set, but pre-select the implied option as the recommended one. Never silently assume — confirm.

### Step 3: Spawn the agent

After getting answers, spawn the agent using:
```
Agent tool with subagent_type: "Infrastructure Documentation Writer"
```

Your prompt to the agent MUST include:
1. Exactly what to document (from user's request + answers)
2. Output destination(s) chosen by user
3. Draft vs live status
4. Which repos were verified on master and which were not
5. **Local markdown location rule**: if local markdown is part of the destination, instruct the agent that files MUST be written to `/var/www/php74/mason/docs/infra/` using the naming convention `YYYY-MM-DD-<slug>.md`. Do NOT write to per-property `<property>/docs/Overview.md` — that path is owned by a different convention (CLAUDE.md per-property overviews). The infra-doc skill always uses the workspace-level `docs/infra/` directory.

And ALWAYS include this Confluence context block verbatim:

---
**Confluence Config (do not ask the user for these — they are pre-configured):**
- Site: `admedia-jira.atlassian.net`
- Space: `EN` (spaceId: `1146884`)
- Root folder ID: `681082881` — this is the DOCUMENTATION ROOT
- All infra docs live under this root as a tree
- For property/integration-level docs: create a PARENT PAGE under root `681082881`, then nest sub-pages under it
- **Search existing pages first** — use CQL to check what already exists. Do NOT duplicate content. Cross-link instead.
- Example hierarchy:
  - Root (681082881)
    - "Click Tracking" (existing page 681115650)
    - "Amazon Ads" (parent page, parentId=681082881)
      - "SSP Incoming Flow"
      - "OpenSearch & RefTags"
    - "Ad Center" (parent page, parentId=681082881)
---

Tell the agent to use AskUserQuestion if it discovers gaps in the codebase that only a human can fill (production-only values, ownership info, etc.).

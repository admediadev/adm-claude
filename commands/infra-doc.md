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

Before spawning the agent, you MUST ask the user using AskUserQuestion. Batch these into one question set:

**If arguments are provided**, infer what you can from them but still ask:
- Output destination: Local only / Confluence only / Both (default: Both)
- Publish status: Draft or Live (default: Draft)

**If no arguments are provided**, also ask:
- What to document: Full platform / Specific property / Specific file or API / Update existing doc

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

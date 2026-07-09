---
name: chgrel
description: "Developer - generate a Jira CHGREL change/release ticket description from this conversation using the new simplified format, then optionally create the ticket after approval. Use for chgrel, release ticket, change ticket, ticket description."
---

Generate a Jira ticket description summarizing the work done in this conversation using the **new simplified CHGREL change/release format**, and optionally create the ticket in the CHGREL Jira project after user approval.

**Format authority:** the template in `CHGREL-1` (latest comments by Sireesha Chilakamarri + Sahil Kamthania) is the source of truth. `CHGREL-1074` is a filled-out reference example. This format is *mandatory* and is designed to force QA evidence into the ticket — see https://admedia-jira.atlassian.net/browse/CHGREL-1.

## Workflow

### 1. Gather Context

Review the full conversation to understand:
- What was changed and why
- Which files were modified
- Which sites/properties were impacted
- Which database tables were affected (if any)
- Who developed, reviewed, or approved the work
- Any PR / commit URLs referenced (Bitbucket: `bit.admedia.com`)
- **Source ticket(s)** — every distinct Jira issue key referenced in the conversation that this change implements (e.g. `PUB-3343`, `MNGT-868`, URLs like `admedia-jira.atlassian.net/browse/XYZ-123`). These become **Included Jira Tickets** and are auto-linked from the new CHGREL ticket in step 5.
- **Dependency ticket(s)** — any WebOPS / NetOPs / Systems / other tickets this code or config depends on (separate from the included/originating tickets).

Also check `git diff` and `git log` for recent changes on the current branch if available.

### 2. Generate Description (new CHGREL format)

Plan the content using these **six numbered sections**. Sections 1–5 go in the ticket description. Section 6 (Production Validation) is posted as a **Jira comment after deployment** — see step 6. When previewing to the user, render in plain markdown for readability; when creating the Jira issue, convert to ADF as described in step 5.

**1. Scope of Change**
- **Change Summary** — 1–3 sentences: what was done and why
- **Included Jira Tickets** — the originating/source ticket(s); link or mention every one
- **Dependency Tickets** — WebOPS/NetOPs/Systems/etc. this depends on, or "None"

**2. Impact & Risk**
- **User Impact** — what users experience from this change
- **Risk Level** — `Low` / `Medium` / `High`
- **Known Risks** — specific risks, or "None"

**3. Deployment & Rollback**
- **Deployment date** — **default to today's date** (do not ask unless the user hints at a future/scheduled window)
- **Rollback Plan** — how to revert (e.g. "Revert PR #424 and redeploy", "Revert commit <hash>")
- **Developer** — default to the user (Mason Leavey) unless the conversation clearly indicates someone else
- **Reviewer** — **default to Sireesha Chilakamarri** unless the user says otherwise

**4. Approval**
- **Business Approval** — **default to the originating/source ticket's reporter** (fetch the `reporter` field of the included Jira ticket). Fall back to "Pending" only if there is no source ticket.
- **Technical Approval** — **default to Sireesha Chilakamarri** unless the user says otherwise

**Defaults & when to ask:** Apply the defaults above silently. Only stop to ask the user when a field is genuinely ambiguous or curious — e.g. no source ticket exists (so no reporter for Reviewer), the deployment looks scheduled rather than immediate, or the Developer/approvers conflict with what the conversation shows. Otherwise fill the defaults and proceed straight to the draft/preview.

**5. QA Validation Requirements** — **mandatory**. For **every Jira ticket included in the release**, list concrete test cases (not just placeholders). Minimums per included ticket:
- At least **3 validated test cases** total
- At least **1 positive** (happy path) test case
- At least **1 negative** test case (invalid input, error handling, permission issues, edge conditions)
- At least **1 regression** test case covering existing functionality impacted by the change
- Add more depending on the radius of impact / risk level
- Note that **screenshots, logs, SQL output, API responses, or links to automation runs** must be attached (these go in the QA comment, step 6)

Write each test case as a specific, runnable step with the expected result (e.g. "Click *Last 7 Days* on the Advertiser Dashboard → start/end = 2026-06-15 / 2026-06-21, today excluded"). Derive them from the actual change and any dev testing already done in the conversation.

**6. Production Validation (Mandatory After Deployment)** — *not* part of the initial description. After the code is deployed, this is added as a **Jira comment** (step 6) and only then may the ticket be closed as Done. Plan its checklist now so the user knows what evidence to capture:
- **Monitoring Completed** — key business workflow verified
- **Logs checked**
- **Error rates checked**
- **Key business workflow verified**
- Attach QA evidence: screenshots, SQL output, log excerpts, API responses

### 3. Review with User

Present the description (sections 1–5) to the user as a plain markdown preview for review. Surface any fields you had to guess or that are still `Pending`/unknown (Developer, Reviewer, approvals, deployment date) and ask the user to confirm or fill them. Ask if any fields need adjustment.

### 4. Offer to Create the Jira Ticket

After the user approves the description, ask:

> Want me to create this as a ticket in the CHGREL project on admedia-jira.atlassian.net? If yes, confirm: summary (short title), issue type (default Task), priority (default P3).

Do not create the ticket unless the user explicitly approves. If they decline, stop — the generated description alone is the deliverable.

### 5. Create the Ticket (only after explicit approval)

If the user approves, use the Atlassian MCP tool `mcp__plugin_atlassian_atlassian__createJiraIssue` with:

- `cloudId`: `admedia-jira.atlassian.net`
- `projectKey`: `CHGREL`
- `issueTypeName`: `Task` (unless the user specified another type — Story, Bug, Epic, Sub-task)
- `summary`: a short title derived from the Change Summary (<= ~120 chars). Ask the user to confirm/edit before submitting.
- `contentFormat`: `adf`
- `description`: a JSON-stringified **ADF document** built from the template below. **Do not pass plain markdown** — Jira's markdown renderer flattens unlabeled section headers into a single wall of text, which is what we are explicitly fixing.
- `additional_fields`: `{"priority": {"name": "P3"}}` by default. Use `P1 Blocker` / `P2 Major` / `P3` / `P4` / `P5` if the user specified otherwise.

#### ADF Document Template

Build the description as a single ADF doc. Use `heading` level 2 for each numbered section title, `bulletList` for lists, and single-line `paragraph`s with a bold label for short labeled fields. Wrap the **Risk Level** line in an ADF `panel` (`panelType`: `info` for Low, `note` for Medium, `warning` for High, `error` for severe production-affecting changes) so it stands out in Jira. Sections 1–5 only; section 6 is a post-deploy comment.

Skeleton (fill the `text` values; keep the structure):

```json
{
  "type": "doc",
  "version": 1,
  "content": [
    { "type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "1. Scope of Change"}] },
    { "type": "paragraph", "content": [
      {"type": "text", "text": "Change Summary: ", "marks": [{"type": "strong"}]},
      {"type": "text", "text": "<summary>"}
    ]},
    { "type": "paragraph", "content": [
      {"type": "text", "text": "Included Jira Tickets: ", "marks": [{"type": "strong"}]},
      {"type": "text", "text": "PUB-3343", "marks": [{"type": "link", "attrs": {"href": "https://admedia-jira.atlassian.net/browse/PUB-3343"}}]}
    ]},
    { "type": "paragraph", "content": [
      {"type": "text", "text": "Dependency Tickets: ", "marks": [{"type": "strong"}]},
      {"type": "text", "text": "None"}
    ]},

    { "type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "2. Impact & Risk"}] },
    { "type": "paragraph", "content": [
      {"type": "text", "text": "User Impact: ", "marks": [{"type": "strong"}]},
      {"type": "text", "text": "<impact>"}
    ]},
    { "type": "panel", "attrs": {"panelType": "info"}, "content": [
      { "type": "paragraph", "content": [
        {"type": "text", "text": "Risk Level: ", "marks": [{"type": "strong"}]},
        {"type": "text", "text": "<Low / Medium / High>"}
      ]}
    ]},
    { "type": "paragraph", "content": [
      {"type": "text", "text": "Known Risks: ", "marks": [{"type": "strong"}]},
      {"type": "text", "text": "<risks or None>"}
    ]},

    { "type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "3. Deployment & Rollback"}] },
    { "type": "paragraph", "content": [
      {"type": "text", "text": "Deployment date: ", "marks": [{"type": "strong"}]},
      {"type": "text", "text": "<YYYY-MM-DD or window>"}
    ]},
    { "type": "paragraph", "content": [
      {"type": "text", "text": "Rollback Plan: ", "marks": [{"type": "strong"}]},
      {"type": "text", "text": "<rollback>"}
    ]},
    { "type": "paragraph", "content": [
      {"type": "text", "text": "Developer: ", "marks": [{"type": "strong"}]},
      {"type": "text", "text": "<name>"}
    ]},
    { "type": "paragraph", "content": [
      {"type": "text", "text": "Reviewer: ", "marks": [{"type": "strong"}]},
      {"type": "text", "text": "<name>"}
    ]},

    { "type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "4. Approval"}] },
    { "type": "paragraph", "content": [
      {"type": "text", "text": "Business Approval: ", "marks": [{"type": "strong"}]},
      {"type": "text", "text": "<name or Pending>"}
    ]},
    { "type": "paragraph", "content": [
      {"type": "text", "text": "Technical Approval: ", "marks": [{"type": "strong"}]},
      {"type": "text", "text": "<name or Pending>"}
    ]},

    { "type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "5. QA Validation Requirements"}] },
    { "type": "bulletList", "content": [
      { "type": "listItem", "content": [{"type": "paragraph", "content": [
        {"type": "text", "text": "Positive: ", "marks": [{"type": "strong"}]},
        {"type": "text", "text": "<happy-path case + expected result>"}
      ]}]},
      { "type": "listItem", "content": [{"type": "paragraph", "content": [
        {"type": "text", "text": "Negative: ", "marks": [{"type": "strong"}]},
        {"type": "text", "text": "<invalid/edge case + expected result>"}
      ]}]},
      { "type": "listItem", "content": [{"type": "paragraph", "content": [
        {"type": "text", "text": "Regression: ", "marks": [{"type": "strong"}]},
        {"type": "text", "text": "<existing-behavior case + expected result>"}
      ]}]}
    ]},
    { "type": "paragraph", "content": [{"type": "text", "text": "Evidence (screenshots, logs, SQL output, API responses, automation links) to be attached in a QA comment before close.", "marks": [{"type": "em"}]}] }
  ]
}
```

Notes when constructing ADF:

- If a section has no content (e.g. Dependency Tickets = None), use a single `paragraph` with text `"None"` instead of an empty `bulletList` — empty lists render awkwardly.
- For URLs inside text (PR links, ticket URLs), use a `text` node with a `link` mark: `{"type": "text", "text": "PUB-3343", "marks": [{"type": "link", "attrs": {"href": "https://admedia-jira.atlassian.net/browse/PUB-3343"}}]}`.
- Pick the Risk Level panel type: `info` for Low, `note` for Medium, `warning` for High, `error` for severe production-affecting changes.
- Pass the ADF as a JSON-stringified value to the `description` parameter.

**Project reference:** CHGREL tickets look like `CHGREL-XXX` and are listed at https://admedia-jira.atlassian.net/browse/CHGREL. Default issue type for change/release tickets is `Task`. Common priorities in use: `P1 Blocker`, `P2 Major`, `P3`.

After the issue is created, **immediately link it to every source/included ticket collected in step 1** using `mcp__plugin_atlassian_atlassian__createIssueLink`. This is the default behavior — do not ask first. For each source ticket key:

- `cloudId`: `admedia-jira.atlassian.net`
- `inwardIssue`: the source ticket key (e.g. `PUB-3343`)
- `outwardIssue`: the newly created CHGREL key (e.g. `CHGREL-690`)
- `type`: `Relates`

If the conversation referenced no source ticket, skip the link step silently. If the user explicitly says "don't link" or "skip the link," skip it. If a link fails (e.g. permission error, missing issue), report the failure but still report the ticket creation as successful.

Report the new ticket key, URL, and any links back to the user (e.g. `Created CHGREL-647: https://admedia-jira.atlassian.net/browse/CHGREL-647 — linked to PUB-3343 (Relates)`).

If ticket creation fails, show the error and ask whether to retry, adjust fields, or stop.

### 6. Post-Deployment QA / Production Validation Comment

The new format **forces QA evidence into the ticket as a comment** — the ticket may only be closed as Done after this is added. This step happens *after the change is actually deployed*, so only do it when the user confirms deployment is done (or asks for it explicitly). Build a comment with `mcp__plugin_atlassian_atlassian__addCommentToJiraIssue` (`cloudId`: `admedia-jira.atlassian.net`, `issueIdOrKey`: the CHGREL key) containing:

- The **executed** QA test cases from section 5 marked pass/fail, each with its evidence (paste SQL output / log lines in an ADF `codeBlock`, attach or reference screenshots).
- **Production Validation** (section 6): Monitoring Completed, Logs checked, Error rates checked, Key business workflow verified — each with a short result line.

Do not add this comment with placeholder/unverified results. If QA evidence does not exist yet in the conversation, tell the user what still needs to be captured and stop.

## Goal

Produce a clean Jira ticket description in the **new simplified CHGREL format** (six sections; Risk Level panel; mandatory positive/negative/regression QA test cases) that renders properly in Jira as ADF — and, when the user approves, create the actual CHGREL ticket, auto-link it to any source tickets, and (after deployment) post the mandatory QA / production-validation comment so the ticket can be closed.

Generate a Jira ticket description summarizing the work done in this conversation, and optionally create the ticket in the CHGREL Jira project after user approval.

## Workflow

### 1. Gather Context

Review the full conversation to understand:
- What was changed and why
- Which files were modified
- Which sites/properties were impacted
- Which database tables were affected (if any)
- Who reviewed or was involved
- Any PR / commit URLs referenced (Bitbucket: `bit.admedia.com`)
- **Source ticket(s)** — any Jira issue keys referenced in the conversation that this change implements (e.g. `MNGT-868`, `INFRA-123`, URLs like `admedia-jira.atlassian.net/browse/XYZ-123`). Collect every distinct issue key. These will be auto-linked from the new CHGREL ticket in step 5.

Also check `git diff` and `git log` for recent changes on the current branch if available.

### 2. Generate Description

Plan the content using these sections (mirrors the CHGREL convention — see CHGREL-638 as a reference). When previewing to the user, render in plain markdown for readability. When creating the Jira issue, convert to ADF as described in step 5.

Required sections:

- **Details** — 1-3 sentences: what was done and why
- **Plan** — short bullet points of what was executed
- **Impact** — None / Low / Medium / High, with brief explanation if not None
- **Sites Impacted** — bullet list of affected sites/endpoints, or "None"
- **Tables Impacted** — bullet list of affected DB tables, or "None"
- **Rollback Plan** — how to revert (e.g. "Revert the release" or "Revert commit <hash>")
- **Proof of Testing Evidence** — any test results from conversation, or "None on dev server"
- **Deployment Date / Time** — today's date or ask the user
- **Developer** — ask the user if not clear from conversation
- **Documented Approval** — reviewer name (from conversation), or ask the user
- **Testing Instructions** — bullet list of steps to verify the change works after deployment

### 3. Review with User

Present the description to the user as a plain markdown preview for review. Ask if any fields need adjustment.

### 4. Offer to Create the Jira Ticket

After the user approves the description, ask:

> Want me to create this as a ticket in the CHGREL project on admedia-jira.atlassian.net? If yes, confirm: summary (short title), issue type (default Task), priority (default P3).

Do not create the ticket unless the user explicitly approves. If they decline, stop — the generated description alone is the deliverable.

### 5. Create the Ticket (only after explicit approval)

If the user approves, use the Atlassian MCP tool `mcp__plugin_atlassian_atlassian__createJiraIssue` with:

- `cloudId`: `admedia-jira.atlassian.net`
- `projectKey`: `CHGREL`
- `issueTypeName`: `Task` (unless the user specified another type — Story, Bug, Epic, Sub-task)
- `summary`: a short title derived from the Details section (<= ~120 chars). Ask the user to confirm/edit before submitting.
- `contentFormat`: `adf`
- `description`: a JSON-stringified **ADF document** built from the template below. **Do not pass plain markdown** — Jira's markdown renderer flattens unlabeled section headers into a single wall of text, which is what we are explicitly fixing.
- `additional_fields`: `{"priority": {"name": "P3"}}` by default. Use `P1 Blocker` / `P2 Major` / `P3` / `P4` / `P5` if the user specified otherwise.

#### ADF Document Template

Build the description as a single ADF doc. Use `heading` level 2 for each section title, `bulletList` for lists, and a single-line `paragraph` with a bold label for short labeled fields (Rollback Plan, Deployment Date / Time, Developer, Documented Approval, Proof of Testing Evidence). Wrap the **Impact** line in an ADF `panel` (`panelType`: `info` for None/Low, `note` for Medium, `warning` for High) so it stands out in Jira.

Skeleton (fill the `text` values; keep the structure):

```json
{
  "type": "doc",
  "version": 1,
  "content": [
    { "type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "Details"}] },
    { "type": "paragraph", "content": [{"type": "text", "text": "<details summary>"}] },

    { "type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "Plan"}] },
    { "type": "bulletList", "content": [
      { "type": "listItem", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "<plan item 1>"}]}] }
    ]},

    { "type": "panel", "attrs": {"panelType": "info"}, "content": [
      { "type": "paragraph", "content": [
        {"type": "text", "text": "Impact: ", "marks": [{"type": "strong"}]},
        {"type": "text", "text": "<Low — explanation>"}
      ]}
    ]},

    { "type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "Sites Impacted"}] },
    { "type": "bulletList", "content": [
      { "type": "listItem", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "<site 1>"}]}] }
    ]},

    { "type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "Tables Impacted"}] },
    { "type": "paragraph", "content": [{"type": "text", "text": "None"}] },

    { "type": "paragraph", "content": [
      {"type": "text", "text": "Rollback Plan: ", "marks": [{"type": "strong"}]},
      {"type": "text", "text": "<rollback>"}
    ]},
    { "type": "paragraph", "content": [
      {"type": "text", "text": "Proof of Testing Evidence: ", "marks": [{"type": "strong"}]},
      {"type": "text", "text": "<evidence>"}
    ]},
    { "type": "paragraph", "content": [
      {"type": "text", "text": "Deployment Date / Time: ", "marks": [{"type": "strong"}]},
      {"type": "text", "text": "<YYYY-MM-DD or window>"}
    ]},
    { "type": "paragraph", "content": [
      {"type": "text", "text": "Developer: ", "marks": [{"type": "strong"}]},
      {"type": "text", "text": "<name>"}
    ]},
    { "type": "paragraph", "content": [
      {"type": "text", "text": "Documented Approval: ", "marks": [{"type": "strong"}]},
      {"type": "text", "text": "<name and source, e.g. Sireesha Chilakamarri (per AO-18732 comment)>"}
    ]},

    { "type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "Testing Instructions"}] },
    { "type": "bulletList", "content": [
      { "type": "listItem", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "<step 1>"}]}] }
    ]}
  ]
}
```

Notes when constructing ADF:

- If a section has no content (e.g. Tables Impacted = None), use a single `paragraph` with text `"None"` instead of an empty `bulletList` — empty lists render awkwardly.
- For URLs inside text (PR links, ticket URLs), use a `text` node with a `link` mark: `{"type": "text", "text": "AO-18732", "marks": [{"type": "link", "attrs": {"href": "https://admedia-jira.atlassian.net/browse/AO-18732"}}]}`.
- Pick the panel type from the impact level: `info` for None/Low, `note` for Medium, `warning` for High, `error` for severe production-affecting changes.
- Pass the ADF as a JSON-stringified value to the `description` parameter.

**Project reference:** CHGREL tickets look like `CHGREL-XXX` and are listed at https://admedia-jira.atlassian.net/browse/CHGREL. Default issue type for change/release tickets is `Task`. Common priorities in use: `P1 Blocker`, `P2 Major`, `P3`.

After the issue is created, **immediately link it to every source ticket collected in step 1** using `mcp__plugin_atlassian_atlassian__createIssueLink`. This is the default behavior — do not ask first. For each source ticket key:

- `cloudId`: `admedia-jira.atlassian.net`
- `inwardIssue`: the source ticket key (e.g. `MNGT-868`)
- `outwardIssue`: the newly created CHGREL key (e.g. `CHGREL-690`)
- `type`: `Relates`

If the conversation referenced no source ticket, skip the link step silently. If the user explicitly says "don't link" or "skip the link," skip it. If a link fails (e.g. permission error, missing issue), report the failure but still report the ticket creation as successful.

Report the new ticket key, URL, and any links back to the user (e.g. `Created CHGREL-647: https://admedia-jira.atlassian.net/browse/CHGREL-647 — linked to MNGT-868 (Relates)`).

If ticket creation fails, show the error and ask whether to retry, adjust fields, or stop.

## Goal

Produce a clean Jira ticket description that renders properly in Jira (proper headings, bullet lists, an info/note/warning panel for the Impact line) by submitting it as ADF — and, when the user approves, create the actual CHGREL ticket and auto-link it to any source tickets referenced in the conversation, in a single step.

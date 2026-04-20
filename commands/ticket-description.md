Generate a Jira ticket description summarizing the work done in this conversation.

## Workflow

### 1. Gather Context

Review the full conversation to understand:
- What was changed and why
- Which files were modified
- Which sites/properties were impacted
- Which database tables were affected (if any)
- Who reviewed or was involved

Also check `git diff` and `git log` for recent changes on the current branch if available.

### 2. Generate Description

Output the ticket description in this exact format:

```
Details

[Brief summary of what was done and why — 1-3 sentences]

Plan

[Short bullet points of what was executed]

Impact - [None / Low / Medium / High — with brief explanation if not None]

Sites Impacted

[List of affected sites/endpoints, or "None"]

Tables Impacted

[List of affected DB tables, or "None"]

Rollback Plan - [How to revert — e.g. "Revert the release" or "Revert commit <hash>"]

Reviewed By - [Names mentioned in conversation, or ask the user]

Proof of Testing Evidence - [Any test results from conversation, or "None on dev server"]

Deployment Date / Time - [Today's date or ask the user]

Developer - [Ask the user if not clear from conversation]

Testing Instructions

[Steps to verify the change works after deployment]
```

### 3. Review

Present the description to the user for review before they paste it into Jira. Ask if any fields need adjustment.

## Goal

Produce a clean, copy-pasteable Jira ticket description summarizing the work done in the current conversation.

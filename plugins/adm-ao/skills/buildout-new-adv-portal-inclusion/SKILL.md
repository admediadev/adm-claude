---
name: buildout-new-adv-portal-inclusion
description: "STUB — Ad Ops · AA-22 · AA-33 AO Portal inclusion request for a new advertiser. Will interview for advertiser name/AID, reporting email subject/sender/receiver + example report, and emit validated JSON to file the Portal Inclusion Jira ticket + Slack alert. Note: AA-33 already ships as a form-fill tab in adops (Manoj); this skill would generate its JSON. Not yet implemented."
---

# adm · ao-buildout-new-adv-portal-inclusion (AA-33) — STUB

**Not yet implemented.** Placeholder for AA-33 (New Advertiser Buildouts, Prompt 5).

**Context:** AA-33 is already live in adops as the **AO Portal Inclusion Factory** form (built by Manoj). This skill would provide the prompt-driven JSON path into that same feature.

**Intended output JSON** will carry: advertiser name, AID, reporting email subject, sender, receiving account, and an example-report reference → adops creates the Portal Inclusion Jira ticket (AO project) + posts the Slack alert.

When implementing: align the JSON to the AO Portal Inclusion form fields, add a `portal_inclusion` type + schema, bundle it here.

---
name: ao-buildout-new-adv-channels
description: "STUB — Ad Ops · AA-22 · AA-30 Compile Slack channels for a new advertiser buildout. Will interview for advertiser/vertical/url/launch/KPI + geoblocks/trademark prefs and emit a validated JSON to create the brand ao-pub Slack channel, auto-invite pubAMs, and post the launch announcement. Not yet implemented."
---

# adm · ao-buildout-new-adv-channels (AA-30) — STUB

**Not yet implemented.** Placeholder for AA-30 (New Advertiser Buildouts, Prompt 2).

**Intended output JSON** (once built) will carry: advertiser name / AID, vertical, url, expected launch date, publisher KPI, and extras (geoblocks, trademark-bidding prefs) → adops creates a brand-specific `ao-pub` Slack channel, auto-invites pubAMs, and posts the boilerplate launch announcement.

When implementing: add a `new_advertiser_channels` type to adops `Config/BuildoutTypes.php`, run `bin/gen-buildout-schema.php new_advertiser_channels`, bundle the schema here, then mirror the `ao-buildout-new-adv-compile` skill's procedure.

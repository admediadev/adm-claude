---
name: ao-buildout-new-adv-tracker
description: "STUB — Ad Ops · AA-22 · AA-31 Create a tracker for a new advertiser. Will interview for advertiser name, SoT (SA360/GA/Other), KPI (CPA/ROAS) and brand/nonbrand split, then emit validated JSON to instantiate the Advertiser Tracker sheet with the right tabs shown/hidden/renamed. Not yet implemented."
---

# adm · ao-buildout-new-adv-tracker (AA-31) — STUB

**Not yet implemented.** Placeholder for AA-31 (New Advertiser Buildouts, Prompt 3).

**Intended output JSON** will carry: advertiser name, source-of-truth, KPI, and whether brand/nonbrand goals are separate → adops copies the tracker template, hides breakdown/summary tabs not matching SoT/KPI, and renames the Day-to-Day + Breakdown tabs to the current month/year.

When implementing: add a `new_advertiser_tracker` type to `Config/BuildoutTypes.php`, regenerate the schema, bundle it here, mirror the compile skill.

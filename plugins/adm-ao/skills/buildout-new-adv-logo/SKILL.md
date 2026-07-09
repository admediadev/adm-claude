---
name: buildout-new-adv-logo
description: "STUB — Ad Ops · AA-22 · AA-32 Format a logo for a new advertiser. Will collect the advertiser ID + logo reference and emit validated JSON to drive formatting to a 200x200 transparent PNG, S3 upload, and Advertiser_info path update. Note: the image bytes are NOT carried in JSON — the logo is a file reference / separate upload. Not yet implemented."
---

# adm · ao-buildout-new-adv-logo (AA-32) — STUB

**Not yet implemented.** Placeholder for AA-32 (New Advertiser Buildouts, Prompt 4).

**Intended output JSON** will carry: advertiser ID + a logo file reference (filename/path) → adops formats the image to a 200×200 transparent PNG, uploads to S3 (`advlogo` bucket), and updates `Advertiser_info.logo`. The binary itself is uploaded via the form's file input, not embedded in JSON.

When implementing: add a `new_advertiser_logo` type to `Config/BuildoutTypes.php`, regenerate the schema, bundle it here, mirror the compile skill.

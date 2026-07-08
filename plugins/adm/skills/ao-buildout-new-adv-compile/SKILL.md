---
name: ao-buildout-new-adv-compile
description: "Ad Ops · AA-22 New Advertiser Buildouts · AA-29 Compile a New Advertiser. Generate a strict, validated buildout JSON to import into the adops Advertiser Factory. Use when the user says 'compile a new advertiser', 'advertiser factory', 'new advertiser buildout', 'generate buildout json', or names buildout type new_advertiser. Interviews for missing fields, applies conditional logic, validates against the bundled JSON Schema, writes the file to the current directory."
---

# adm · ao-buildout-new-adv-compile (AA-29)

Produces the JSON that the adops Advertiser Factory imports. adops stays a simple form + "Import JSON"; this skill is the prompt layer: it interviews the user, applies the boolean logic, validates strictly, and emits a portable file.

## Locate the bundled schema (single source of truth)
The JSON Schema ships inside this plugin. Resolve its path at runtime — do not hardcode:
```bash
TYPE="new_advertiser"   # the requested buildout type
SCHEMA="$(find "${CLAUDE_PLUGIN_ROOT:-$HOME/.claude/plugins}" -name "buildout-$TYPE.schema.json" 2>/dev/null | head -1)"
echo "$SCHEMA"
```
If `$SCHEMA` is empty, the type has no bundled schema — stop and tell the user which types are available (list the `schemas/buildout-*.schema.json` files found).

## Procedure
1. **Determine `buildout_type`** (default `new_advertiser`) and locate its schema (above). **Read the schema** for the authoritative field list, types, enums, and required/conditional rules — never invent field names.
2. **Collect field values**: use whatever the user passed as parameters; ask concisely (grouped) for any missing **required** fields. Do not guess values.
3. **Apply the conditional logic while collecting:**
   - Values use the form's option keys exactly (the schema enums are generated from them): `has_agency` is `"yes"`/`"no"` (not a boolean); booleans like `enable_xml` are `"1"`/`"0"`.
   - `source_of_truth == "SA360"` → leave `link_prefix`/`link_suffix` blank (SA360 supplies links later); for `GA`/`Other` provide `link_prefix`.
   - `has_agency == "yes"` → `agency_name` is required.
   - `categories` → array of strings; one starting campaign per category downstream.
   - `keywords` → optional `brand`/`exact`/`negative` string arrays.
   - Dates → `YYYY-MM-DD`; `display_url` must start with `http(s)://`.
4. **Assemble** to match the schema's shape: `{ "schema_version": "1.0", "buildout_type": <type>, "main": { …essentials… }, "sections": { "advertiser_profile": {…}, "campaign": {…}, "economics": {…}, "creative_link": {…}, "targeting": {…}, "demographics": {…}, "keywords": {…} } }`. Put the required essentials in `main`; only include a section (and only the fields the user gave) when there's data — every section is optional. Read the schema for the exact field list per section.
5. **Validate before writing** — never emit an invalid/partial file:
   ```bash
   python3 - "$SCHEMA" "$OUT" <<'PY'
   import json,sys
   import jsonschema                       # falls back to manual checks if unavailable
   schema=json.load(open(sys.argv[1])); doc=json.load(open(sys.argv[2]))
   jsonschema.validate(doc,schema); print("valid")
   PY
   ```
   If `jsonschema` isn't installed, validate by hand against every rule in the schema (required, enums, formats, the `if/then` conditionals). Fix any error and re-validate.
6. **Write** the file to the **current working directory** as `<advertiser-slug>-<YYYYMMDD>.json` (short kebab slug of the advertiser name). Print the path.
7. **Tell the user** to upload it in adops → *Adv Factory via Automation → Compile New Advertiser → Import JSON*; adops re-validates against the same schema and prefills the form for review.

## Rules
- Exact field names from the schema; no extras (`additionalProperties:false` on the root, `main`, and each section).
- One buildout per file.
- Everything type-specific comes from the bundled schema, not hardcoded here. The schema is generated from adops `Config/BuildoutTypes.php` via `bin/gen-buildout-schema.php` — never hand-edit it.

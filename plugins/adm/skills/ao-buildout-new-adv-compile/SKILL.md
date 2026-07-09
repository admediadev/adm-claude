---
name: ao-buildout-new-adv-compile
description: "Ad Ops · AA-22 New Advertiser Buildouts · AA-29 Compile a New Advertiser. Generate a strict, validated buildout JSON to import into the adops Advertiser Factory. Use when the user says 'compile a new advertiser', 'advertiser factory', 'new advertiser buildout', 'generate buildout json', or names buildout type new_advertiser. Interviews for the main essentials AND each per-step section sub-form (offering to skip any and finish it in the portal), applies conditional logic, validates against the bundled JSON Schema, writes the file, and explains how to import it into adops and finish/run the steps."
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

## Interview rules (apply throughout)
- **Always label each field `(required)` or `(optional)`** when you ask for it, and show its allowed values for enums (e.g. `source_of_truth (required): SA360 | GA | Other`).
- **Always let the user fill or skip.** Optional fields (and whole sections) can be skipped — say so explicitly ("…or reply *skip*"). Required fields should be answered, but the user may still defer; the gap-check (step 6) catches anything required that's still missing before the file is written.
- **Derive "required" from the schema, not memory:** the required set = `main.required` **plus** any field pulled in by an `if/then` conditional (e.g. `agency_name` becomes required when `has_agency` = `yes`). Sections currently have no required fields — they're all optional.

## Procedure
1. **Determine `buildout_type`** (default `new_advertiser`) and locate its schema (above). **Read the schema** for the authoritative field list, types, enums, and required/conditional rules — never invent field names. The schema has two parts: `main` (a few required essentials) and `sections` (per-step sub-forms — all optional).

2. **Interview for `main`** — go field by field, each tagged `(required)`/`(optional)` with its allowed values. Use whatever the user passed as parameters; ask for the rest. Never guess.

3. **Then interview each section sub-form — do NOT stop after `main`.** The sections mirror the adops per-step sub-forms: `advertiser_profile`, `campaign`, `economics`, `creative_link`, `targeting`, `demographics`, `keywords`. Go **one section at a time**:
   - Say what the section is and list its fields (from the schema), then ask the user to provide the values they have — or reply **"skip"** to leave that section for the adops portal.
   - Collect only the values given; drop blanks. Every section is optional, so never block — but always *offer* each one so the JSON can be as complete as the user wants.
   - Up front, offer the shortcut: *"Want to fill all the step details now, or just the essentials and finish the rest in the portal?"* Respect the answer.

4. **Value formats / conditional rules** (apply as you collect):
   - Use the form's option keys exactly (schema enums are generated from them): `has_agency` is `"yes"`/`"no"` (not boolean); booleans like `enable_xml` are `"1"`/`"0"`.
   - `source_of_truth == "SA360"` → leave `link_prefix`/`link_suffix` blank (SA360 supplies links later); for `GA`/`Other` provide `link_prefix`.
   - `has_agency == "yes"` → `agency_name` is required.
   - `categories`, `keywords.brand/exact/negative` → arrays of strings. Dates → `YYYY-MM-DD`; `display_url` starts with `http(s)://`.

5. **Assemble** to the schema shape: `{ "schema_version": "1.0", "buildout_type": <type>, "main": {…}, "sections": { <only sections the user filled> } }`. Include a section only if it has ≥1 value.

   **Required gap-check (do this before writing).** Recompute the required set (`main.required` + conditional requireds like `agency_name` when `has_agency=yes`) and list any that are still **missing or blank**, e.g.:
   > ⚠️ Still need these required fields before I can generate the file: **display_url**, **source_of_truth**. Please provide them (or say *cancel*).

   Ask for each missing required field and fill it in. Repeat until none remain. Only proceed to validate/write once every required field has a value — if the user can't provide one, warn that adops will reject the JSON and the buildout can't be created without it, and stop rather than writing an invalid file. (Optional fields that were skipped are fine to leave out — they can be filled in the portal.)

6. **Validate before writing** — never emit an invalid/partial file:
   ```bash
   python3 - "$SCHEMA" "$OUT" <<'PY'
   import json,sys
   import jsonschema
   schema=json.load(open(sys.argv[1])); doc=json.load(open(sys.argv[2]))
   jsonschema.validate(doc,schema); print("valid")
   PY
   ```
   If `jsonschema` isn't installed, validate by hand against every schema rule (required, enums, formats, `if/then`). Fix and re-validate.

7. **Write** the file to the **current working directory** as `<advertiser-slug>-<YYYYMMDD>.json`. Print the path.

8. **Report next steps explicitly** (always print this, tailored to what was filled):
   - The saved file path, and a one-line summary of which sections were filled vs left for the portal.
   - **Upload:** *"In adops, go to **Adv Factory via Automation → Compile New Advertiser**, click **⬆ Import**, and choose this file. adops re-validates it and opens the buildout with your values prefilled."*
   - **Finish in the portal:** *"Any section you skipped, fill it there: open the buildout, expand the step, click **Edit**, complete its sub-form, and **Save**."*
   - **Run:** *"Then run each step top-to-bottom; the final step (**Create in MNGT**) asks for confirmation before it writes live records."*

## Rules
- Exact field names from the schema; no extras (`additionalProperties:false` on the root, `main`, and each section).
- One buildout per file.
- Everything type-specific comes from the bundled schema, not hardcoded here. The schema is generated from adops `Config/BuildoutTypes.php` via `bin/gen-buildout-schema.php` — never hand-edit it.

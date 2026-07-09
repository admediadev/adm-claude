---
name: buildout-add-type
description: "STUB — Developer · scaffold a new Advertiser Factory buildout type. Will add a type entry to adops Config/BuildoutTypes.php (main + sections + steps), run bin/gen-buildout-schema.php to emit the JSON Schema, bundle it into the adm plugin, and create the matching ao-* skill stub. Establishes the adm-dev-* family. Not yet implemented."
---

# adm · dev-buildout-add-type (developer) — STUB

**Not yet implemented.** First entry in the developer (`dev-*`) family — internal tooling, not for Ad Ops end users.

**Intended behavior:** given a new buildout task, scaffold it end-to-end for a developer:
1. Add the type to adops `html/app/Config/BuildoutTypes.php` (`main` + `sections` + `steps`).
2. Run `php bin/gen-buildout-schema.php <type>` to generate + bundle the JSON Schema.
3. Create the matching `ao-buildout-<...>` skill stub in this plugin.

This keeps `BuildoutTypes.php` the single source of truth and the schema always generated.

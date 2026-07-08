# adm — AdMedia factory skills (Claude Code plugin)

Prompt-driven skills that interview you for inputs and emit a **strict, schema-validated JSON** you upload into the adops portal, which re-validates it and prefills the form. Two families:

- **`ao-*`** — Ad Ops workflows (AA-22 New Advertiser Buildouts, etc.)
- **`dev-*`** — internal developer tooling

## Install
```
/plugin marketplace add https://bit.admedia.com/scm/ad/claude.git
/plugin install adm@admedia
```

## Skills (invoked as slash commands)
| Skill | Invoke | Status |
|---|---|---|
| Compile a new advertiser (AA-29) | `/adm:ao-buildout-new-adv-compile` | ✅ ready |
| Slack channels (AA-30) | `/adm:ao-buildout-new-adv-channels` | 🚧 stub |
| Tracker (AA-31) | `/adm:ao-buildout-new-adv-tracker` | 🚧 stub |
| Logo format (AA-32) | `/adm:ao-buildout-new-adv-logo` | 🚧 stub |
| AO Portal inclusion (AA-33) | `/adm:ao-buildout-new-adv-portal-inclusion` | 🚧 stub |
| CHGREL ticket (dev) | `/adm:dev-chgrel` | ✅ ready |
| drest — prod→dev data (dev) | `/adm:dev-drest` | ✅ ready |
| Infra docs (dev) | `/adm:dev-infra-doc` | ✅ ready |
| Codebase review (dev) | `/adm:dev-codebase-reviewer` | ✅ ready |
| Add a buildout type (dev) | `/adm:dev-buildout-add-type` | 🚧 stub |

## Use (the ready one)
```
/adm:ao-buildout-new-adv-compile
```
Answer the prompts (or pass values in your message). The skill validates against the bundled JSON Schema and writes `<advertiser>-<date>.json` to your current folder. Upload it in adops → **Adv Factory via Automation → Compile New Advertiser → Import JSON**.

## Conventions
- Naming: `adm` (plugin) → `<family>-<domain>-<task>`, e.g. `ao-buildout-new-adv-compile`.
- The JSON contract per type ships at `skills/<skill>/schemas/buildout-<type>.schema.json` and is **generated** from adops `Config/BuildoutTypes.php` via `bin/gen-buildout-schema.php` — never hand-edited (one source of truth, zero drift).
- Update/remove: `/plugin marketplace update admedia` · `/plugin uninstall adm@admedia`.

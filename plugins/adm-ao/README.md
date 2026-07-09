# ao — AdMedia Ad Ops factory skills

Prompt-driven skills that interview you for inputs and emit a **strict, schema-validated JSON** you upload into the adops portal, which re-validates it and prefills the form. Safe for Ad Ops (non-technical) users.

## Install
```
/plugin marketplace add https://bit.admedia.com/scm/ad/claude.git
/plugin install adm-ao@admedia
```

## Skills (AA-22 New Advertiser Buildouts)
| Skill | Invoke | Status |
|---|---|---|
| Compile a new advertiser (AA-29) | `/adm-ao:buildout-new-adv-compile` | ✅ ready |
| Slack channels (AA-30) | `/adm-ao:buildout-new-adv-channels` | 🚧 stub |
| Tracker (AA-31) | `/adm-ao:buildout-new-adv-tracker` | 🚧 stub |
| Logo format (AA-32) | `/adm-ao:buildout-new-adv-logo` | 🚧 stub |
| AO Portal inclusion (AA-33) | `/adm-ao:buildout-new-adv-portal-inclusion` | 🚧 stub |

## Use
```
/adm-ao:buildout-new-adv-compile
```
Answer the prompts (each field is tagged required/optional; you can skip optional ones and finish them in the portal). The skill writes `<advertiser>-<date>.json` to your current folder — upload it in adops → **Adv Factory via Automation → Compile New Advertiser → Import JSON**.

The JSON contract per type ships at `skills/<skill>/schemas/buildout-<type>.schema.json`, generated from adops `Config/BuildoutTypes.php` (one source of truth, zero drift).

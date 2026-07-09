# dev — AdMedia developer tools

Internal engineering skills. Assumes the AdMedia environment (repos, `drest`, VPN/creds).

## Install
```
/plugin marketplace add https://bit.admedia.com/scm/ad/claude.git
/plugin install adm-dev@admedia
```

## Skills
| Skill | Invoke | Status |
|---|---|---|
| CHGREL change/release ticket | `/adm-dev:chgrel` | ✅ ready |
| drest — prod→dev data clone | `/adm-dev:drest` | ✅ ready |
| Infrastructure docs | `/adm-dev:infra-doc` | ✅ ready |
| Multi-agent codebase review | `/adm-dev:codebase-reviewer` | ✅ ready |
| Scaffold a new buildout type | `/adm-dev:buildout-add-type` | 🚧 stub |

Devs typically install **both** `adm-ao@admedia` (to test the Ad Ops flows) and `adm-dev@admedia`.

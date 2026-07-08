---
name: dev-codebase-reviewer
description: "Developer - run a parallel multi-agent, security-focused code review of an AdMedia property. Use for codebase review, review this property, multi-agent review, security audit of the repo."
---

Run a parallel multi-agent code review of an AdMedia property, focused on security vulnerabilities and code quality. Output goes to `<workspace_root>/docs/review-codebase/<repo>/<perspective>.md`.

User request: $ARGUMENTS

## Invocation

```
/admedia-codebase-reviewer <repo>
```

Where `<repo>` is the folder name of the property inside the workspace (not an absolute path), e.g.:

- `/admedia-codebase-reviewer api.admedia.com`
- `/admedia-codebase-reviewer advertisers7_new.admedia.com`
- `/admedia-codebase-reviewer click`

If no repo is given, ask the user which one to scan. Suggest public-facing properties first (highest blast radius):
- `advertisers7_new.admedia.com` (advertiser portal)
- `permissions.admedia.com` (auth / access control)
- `api.admedia.com` (public API)
- `click` (public click tracking)
- `adops.admedia.com`
- `admediacrons.com` (batch jobs touching prod data)
- `adcenter.admedia.com` (CI4 advertiser dashboard)

## Your job (orchestrator)

You are the orchestrator. Do NOT review code yourself — resolve the workspace, sync the repo, then spawn review agents in parallel. Their reports become the deliverable.

---

## Hard rules

### READ-ONLY on production — non-negotiable

This skill runs on production servers. It **observes and reports**. It does not fix, change, or touch anything on the host beyond its own purpose-built output directory.

**Allowed writes (exhaustive list — nothing else):**
- Create the output directory `<workspace_root>/docs/review-codebase/<repo>/` and the Markdown files inside it.
- `git fetch origin --quiet` inside the reviewed repo (updates local refs, does not alter working tree).
- `git pull --ff-only origin master` inside the reviewed repo — **only after explicit user approval in the current conversation** and **only when** the repo is on `master`, the working tree is clean, and the local branch is strictly behind origin (fast-forward possible). Never `--rebase`, never force, never on a non-master branch.

**Forbidden — even if an agent is "confident" it's the right fix:**
- Editing, creating, renaming, `chmod`-ing, or deleting any file inside any reviewed repo.
- Any git mutation other than the single `pull --ff-only` above: no `commit`, `add`, `push`, `checkout`, `switch`, `reset`, `rebase`, `merge`, `stash`, `tag`, `branch -d/-D`, `gc`, `clean`. Read-only `git log`, `git blame`, `git diff`, `git show`, `git rev-parse`, `git status`, `git ls-files` are fine.
- Running the reviewed code. No `php <file>`, no `curl` to prod endpoints, no `composer install`, no `npm`, no build steps, no unit tests, no DB queries against any database. Static inspection only.
- Restarting / killing / signalling processes or services (`systemctl`, `kill`, `pkill`, `service`, `supervisorctl`).
- Writing to Redis, Memcache, MySQL, S3, or any stateful store.
- Opening PRs, filing Jira tickets, editing Confluence pages, posting to Slack, or any other outbound write to external systems. The deliverable is the Markdown under the output directory; humans decide what to do with it.
- Invoking package/host mutators: `apt`, `yum`, `brew`, `pip`, `composer require`, `npm install`, etc.

If an agent says "I could fix this by editing line 332" — put that in its **Recommendation**, verbatim. A human decides whether to act. Propose in text; never execute.

### Ground findings in the actual code, not pre-generated docs

Every agent MUST base its findings on the current source tree at the reviewed commit. Do **not** rely on:
- Prior review files under `docs/reviews/` or `docs/review-codebase/`.
- `docs/solutions/`, `ONBOARDING.md`, `CLAUDE.md`, or any `Overview.md` as a source of truth about code behaviour.
- Cached memory of previous runs.

These may be stale or plain wrong. They are fine as **orientation** ("where's the login handler?") but every claimed finding must cite a concrete `file:line` the agent has read in this run. If a finding cannot be pinned to a file and line, it doesn't ship.

---

## Steps

### Step 1: Resolve the workspace root and repo path

Never hardcode a workspace path — production and dev have different layouts. Resolve `<workspace_root>` in this order and use the first that works:

1. `$ADMEDIA_WORKSPACE_ROOT` env var, if set and a directory exists there.
2. The current working directory (`pwd`) — if `pwd/<repo>/.git` exists, treat `pwd` as the workspace root.
3. Walk up from `pwd` looking for a directory that contains `<repo>/.git` as a direct child (stop at `/` or at the user's home). This handles invocation from inside one of the sibling repos.
4. Otherwise, ask the user: "Can't auto-detect the workspace root. What's the absolute path of the directory that contains `<repo>/`?" — do not guess.

Then set `<repo_path> = <workspace_root>/<repo>` and confirm it's a git repo:

```bash
REPO="<repo>"
ROOT="<workspace_root>"         # resolved above
REPO_PATH="$ROOT/$REPO"
test -d "$REPO_PATH/.git" || { echo "not a git repo: $REPO_PATH" >&2; exit 1; }
```

If `<repo>` happens to be given as an absolute path (contains a leading `/`), accept it as `<repo_path>` directly and derive `<workspace_root>` as its parent directory.

Never fall back to any other hardcoded root. If resolution fails, ask.

### Step 2: Verify branch and sync with origin

Production runs `master`. Do all of the following (read-only first, mutation last and only with approval):

```bash
git -C "$REPO_PATH" status --porcelain
git -C "$REPO_PATH" branch --show-current
git -C "$REPO_PATH" fetch origin --quiet                           # allowed: refs only
git -C "$REPO_PATH" rev-list --left-right --count HEAD...origin/master   # "behind<TAB>ahead"
git -C "$REPO_PATH" log -1 --format='%h %s (%ar)' HEAD
git -C "$REPO_PATH" log -1 --format='%h %s (%ar)' origin/master
```

Decision tree:

| State | Action |
|-------|--------|
| On `master`, clean, up to date with `origin/master` | Proceed. Record `production_parity: in-sync`. |
| On `master`, clean, **behind** `origin/master` (ahead=0) | **Ask the user** for approval to run `git -C "$REPO_PATH" pull --ff-only origin master`. If approved, run it, then re-read the commit sha. Record `production_parity: synced-during-run`. If declined, proceed against the current commit and record `production_parity: behind-origin (<N> commits)`. |
| On `master`, clean, **ahead** of `origin/master` (ahead>0) | Stop. Tell the user master has local unpushed commits. Ask whether to review this state anyway or cancel. Do NOT pull. |
| On `master`, dirty working tree | Warn — agents review committed code only. Ask whether to proceed or cancel. Never stash for them. |
| Not on `master` | Stop. Tell the user the current branch. Offer three choices: (a) wait while they switch to master themselves, (b) review the current branch anyway (record `production_parity: deviated (<branch>)` and include the caveat in every agent prompt), (c) cancel. Never run `git checkout` for them. |

After Step 2, lock in three values for the rest of the run:
- `<branch>` — current branch name.
- `<commit_sha>` — from `git rev-parse HEAD` (short form for human display).
- `<production_parity>` — one of the strings above.

### Step 3: Prepare the output directory

```bash
OUT="$ROOT/docs/review-codebase/$REPO"
mkdir -p "$OUT"
```

If `$OUT/overview.md` already exists from a prior run, ask the user: overwrite, write this run as `overview-YYYY-MM-DD.md` (and per-perspective files get the same dated suffix), or cancel. Never silently overwrite.

### Step 4: Profile the codebase (scoping hints only — NOT the review)

Run in parallel. These numbers help size the review and give agents a starting map. They are **not** findings.

```bash
# Size
find "$REPO_PATH" -type f -name '*.php' ! -path '*/vendor/*' ! -path '*/node_modules/*' ! -path '*/.git/*' | wc -l

# Webroot / entry points
find "$REPO_PATH" -maxdepth 4 -type f \( -name 'index.php' -o -name 'router.php' \) ! -path '*/vendor/*' | head -40

# Danger-signal pointer grep (orientation for agents, not a finding)
grep -rEn --include='*.php' --include='*.inc' --exclude-dir=vendor --exclude-dir=node_modules --exclude-dir=.git \
  -e 'mysql_query|mysqli_query\s*\(.*\$_(GET|POST|REQUEST|COOKIE)' \
  -e '\b(eval|assert|system|exec|shell_exec|passthru|popen|proc_open|unserialize)\s*\(' \
  -e 'move_uploaded_file|\$_FILES' \
  -e 'include(_once)?\s*\(\s*\$|require(_once)?\s*\(\s*\$' \
  -e 'echo\s+\$_(GET|POST|REQUEST|COOKIE)' \
  "$REPO_PATH" 2>/dev/null | wc -l
```

Record the file count and a short list of entry-point paths (as paths **relative to `$REPO_PATH`** — never absolute). Pass them into every agent prompt. Agents still do the real walk — these are just breadcrumbs.

### Step 5: Spawn review agents IN PARALLEL

Send a SINGLE message with MULTIPLE `Agent` tool calls so they run concurrently.

**Baseline agents (always run — 9 perspectives):**

| Perspective | subagent_type | Output file |
|-------------|---------------|-------------|
| Security Sentinel — OWASP Top 10, secrets, auth/authz | `compound-engineering:review:security-sentinel` | `security-sentinel.md` |
| Security Reviewer — auth middleware, input handling, permission checks | `compound-engineering:review:security-reviewer` | `security-reviewer.md` |
| Adversarial Reviewer — actively break auth/payments/data mutations | `compound-engineering:review:adversarial-reviewer` | `adversarial-reviewer.md` |
| Data Integrity Guardian — SQL injection, transactions, migration safety | `compound-engineering:review:data-integrity-guardian` | `data-integrity.md` |
| Correctness Reviewer — logic bugs, edge cases, error propagation | `compound-engineering:review:correctness-reviewer` | `correctness.md` |
| Reliability Reviewer — error handling, timeouts, retries, background jobs | `compound-engineering:review:reliability-reviewer` | `reliability.md` |
| Maintainability Reviewer — dead code, coupling, obscure naming | `compound-engineering:review:maintainability-reviewer` | `maintainability.md` |
| Performance Oracle — slow queries, N+1, memory, scalability | `compound-engineering:review:performance-oracle` | `performance.md` |
| Pattern Recognition — anti-patterns, duplicated logic, inconsistent conventions | `compound-engineering:review:pattern-recognition-specialist` | `patterns.md` |

**Conditional agents (add when the repo matches):**

| Condition | Perspective | subagent_type | Output file |
|-----------|-------------|---------------|-------------|
| Repo exposes HTTP routes (API property, or has `api/` / `routes/` directory) | API Contract Reviewer — request/response shape, versioning, breaking changes | `compound-engineering:review:api-contract-reviewer` | `api-contract.md` |
| Repo has migration / schema-change files (`migrations/`, `*.sql`, `schema.rb`) | Data Migration Expert — migration safety, backfill correctness | `compound-engineering:review:data-migration-expert` | `migration-safety.md` |
| Repo handles payments, billing, or money (dirs matching `billing`, `payment`, `invoice`, `charge`) | Adversarial Reviewer, payments-focused (second invocation) | `compound-engineering:review:adversarial-reviewer` | `adversarial-payments.md` |
| User explicitly asks for an architectural pass | Architecture Strategist — design integrity, structural issues | `compound-engineering:review:architecture-strategist` | `architecture.md` |
| User explicitly asks for a simplification pass | Code Simplicity — YAGNI, over-abstraction, dead branches | `compound-engineering:review:code-simplicity-reviewer` | `simplicity.md` |

Before launching, decide the final agent roster based on the repo's contents (grep for `api/`, `migrations/`, `payment`/`billing` directories) and any explicit asks in `$ARGUMENTS`. Tell the user which perspectives you're running before spawning.

### Step 6: Agent prompt template

Use this exact shape for every agent. Fill placeholders per agent. All file paths passed to the agent should be the **absolute resolved paths** from Step 1 — do not embed any path the orchestrator didn't resolve in this run.

> You are performing a **read-only** code review of the PHP codebase at `<repo_path>` (repo: `<repo>`).
>
> **Commit under review:** `<commit_sha_short>` (`<branch>`, production parity: `<production_parity>`).
> **This runs on a production host.** Do NOT edit, create, rename, delete, chmod, or otherwise touch any file in the reviewed repo. Do NOT run the code. Do NOT query databases. Do NOT make network calls. Static inspection only. If you notice something that needs fixing, write it in your **Recommendation** — a human will decide.
>
> **Ground findings in actual code.** Every finding must cite `file.php:<line>` from a file you read in this run. Do not cite prior review docs, Overview files, or memory — read the current source. Orientation docs are fine as a map; they are not a source of truth about behaviour.
>
> **Your focus (this perspective only):** <one-sentence focus — from the agent's description>.
>
> **PHP-specific signals to weigh heavily** (non-exhaustive, drop any that don't fit this perspective):
> - SQL injection: string-concatenated queries, unparameterized `mysql_query`/`mysqli_query`/`->query()` against user input, raw input in `WHERE`/`ORDER BY`/`LIMIT`/`LIKE`.
> - Dangerous sinks: `eval`, `assert` with a string, `system`, `exec`, `shell_exec`, `passthru`, `popen`, `proc_open`, `unserialize` on user input, `include`/`require` with variable paths.
> - File-upload weaknesses: missing MIME/extension validation, uploads writable inside the webroot, unrestricted `move_uploaded_file`, no size/type allowlist.
> - XSS: unescaped output of user input, `echo $_GET[...]`, unescaped view rendering, server-rendered JSON inlined into HTML.
> - CSRF: state-changing endpoints without a token check.
> - Auth / session: hardcoded credentials, weak session handling, missing authorization checks, IDOR on numeric IDs, session fixation, insecure cookies.
> - Secrets: API keys, DB passwords, tokens in code or logs. **Redact any secret you quote to the last 4 characters**.
> - SSRF / open redirect: user-controlled URLs passed to `curl`/`file_get_contents`/`header('Location:')`.
>
> **Scoping hints** (starting map only — walk beyond these):
> - PHP file count: `<count>`
> - Entry points found (relative to repo root): `<paths>`
> - Public-facing directories seen: `<webroot/httpdocs/public/html>`
>
> **Output:** write your findings to `<OUT>/<your-filename>` as Markdown with this EXACT structure. Use **paths relative to `<repo_path>`** in every `Location:` — never absolute paths.
>
> ```markdown
> ---
> perspective: <your perspective slug>
> repo: <repo>
> branch: <branch>
> commit: <commit_sha_short>
> production_parity: <production_parity>
> reviewed_at: <YYYY-MM-DD>
> ---
>
> # <Perspective Name> — <repo>
>
> ## Summary
> One paragraph: overall posture for this perspective, rough count of findings by priority, biggest theme.
>
> ## Findings (ordered by priority)
>
> ### 1. <Short title>
> - **Priority:** Critical | High | Medium | Low
> - **Impact:** what breaks, which systems/users are affected, blast radius (one or two sentences — be concrete, name the system)
> - **Location:** `relative/path/to/file.php:123` (relative to repo root; add more lines if the issue spans a block)
> - **Evidence:**
>   ```php
>   // 6–15 lines max, the exact code that demonstrates the issue
>   ```
> - **Why this matters:** explain the failure mode or exploit path in 2–4 sentences
> - **Recommendation:** concrete fix. If small, show the corrected code. If larger, describe the shape of the fix and the files it touches.
> - **Effort:** small (≤ 1 hr) | medium (≤ 1 day) | large (> 1 day or design work)
>
> ### 2. <Next finding>
> ...
>
> ## What looked fine
> 2–5 bullets on defensive patterns worth keeping — helps reviewers encourage the right habits.
>
> ## Coverage notes
> - Directories walked: <list, relative to repo root>
> - Directories skipped and why: <list> (e.g., `vendor/` third-party, `node_modules/` third-party)
> - Files read in full vs. spot-checked: rough split
> - Confidence in this report: High | Medium | Low, with one-sentence reason.
> ```
>
> **Priority rubric** (use consistently):
> - **Critical** — remotely exploitable without auth, data loss, secrets exposure, or money-moving bug. Patch today.
> - **High** — exploitable with minor preconditions, strong correctness or data integrity risk, or a compliance concern. Patch this sprint.
> - **Medium** — real defect or risk that needs fixing but has mitigating factors (authenticated only, rarely hit path, limited data).
> - **Low** — hardening opportunity, maintainability concern, or low-impact correctness issue.
>
> **Rules for your output:**
> - Do NOT modify any file. Do NOT run any command that writes to the reviewed repo. Do NOT execute the code. Do NOT hit any database or network service.
> - Every finding needs a real `file:line` you read in this run. No speculative or "likely exists somewhere" findings.
> - Redact secrets to last 4 chars.
> - Paths are **relative to the repo root**, never absolute — keeps reports portable across dev and production hosts.
> - If a finding spans many files (shared pattern), pick the worst representative for **Location** and list the others in an "Also in:" line under **Evidence**.
> - Cap total findings at ~25. If you see more, group the long tail under a single finding and list them as a bulleted table inside that finding.
> - If you walk into a dir you won't review (e.g. `vendor/`), note it under **Coverage notes** — don't pad findings with third-party code.

Add perspective-specific addenda to the focus line and signals list as appropriate. Example: for `api-contract-reviewer`, focus on "route shape, request/response contracts, versioning, breaking changes" and drop the XSS/CSRF bullets.

For the `adversarial-payments` invocation, add: "Focus exclusively on payment/billing paths. Try to construct scenarios that double-charge, skip charges, leak card data, or bypass refund/void controls."

### Step 7: Synthesize `overview.md`

After all agents return, the orchestrator writes `<OUT>/overview.md`. Do NOT delegate this to an agent — you saw all the reports, you do the merge.

```markdown
---
repo: <repo>
branch: <branch>
commit: <commit_sha_short> — <subject>
reviewed_at: <YYYY-MM-DD>
production_parity: <in-sync | synced-during-run | behind-origin | deviated | dirty>
perspectives_run: [<list of agents run>]
---

# Codebase Review — <repo>

## At a glance
- PHP files scanned: <count>
- Perspectives run: <N>
- Findings: Critical <N> · High <N> · Medium <N> · Low <N>
- **Top 3 risks:** one line each, each ending with `relative/file.php:LINE → [perspective](./perspective.md)`

## Highest-priority fixes (de-duplicated)
Numbered list, most urgent first. Merge duplicates where two perspectives flagged the same `file:line`. For each item:
- **Priority**
- **Impact** (one sentence)
- **Location** `relative/path/to/file.php:LINE`
- **Recommendation** (one sentence)
- **Effort**
- Source perspective(s): linked

## Themes across perspectives
2–5 bullets on patterns that showed up across multiple reports. Examples: "Unparameterized queries in at least 4 controllers (flagged by Security Sentinel + Data Integrity)", "File-upload handlers share a missing extension allowlist (Security Reviewer + Adversarial)".

## Perspective reports
- [Security Sentinel](./security-sentinel.md) — OWASP Top 10 / secrets / auth
- [Security Reviewer](./security-reviewer.md) — auth middleware & input handling
- [Adversarial Reviewer](./adversarial-reviewer.md) — failure scenarios
- [Data Integrity](./data-integrity.md) — SQL injection, transactions
- [Correctness](./correctness.md) — logic bugs
- [Reliability](./reliability.md) — error handling, timeouts
- [Maintainability](./maintainability.md) — dead code, coupling
- [Performance](./performance.md) — queries, N+1, memory
- [Patterns](./patterns.md) — anti-patterns, duplication
<plus any conditional perspectives that ran>

## What to do next
Short, concrete:
- Which perspective's top finding should be ticketed first, and why.
- Whether any finding blocks a release / deploy.
- Suggested next repo to review (based on exposure / coverage gaps).
- Anything the orchestrator noticed during sync (behind origin, deviated branch, dirty tree) that a human needs to act on.
```

De-dupe when building "Highest-priority fixes": if two perspectives flag the same `file:line`, keep one entry and list all source perspectives under it.

### Step 8: Report back to the user

End your reply with:
- Path to `overview.md` (the resolved absolute path from this run).
- Count of Critical/High findings.
- Production-parity summary (did we sync during the run? any deviation caveats?).
- One sentence on whether this repo looks materially worse or better than expected.
- Offer to run the same command against the next repo on the public-facing priority list.

---

## Guardrails

- **Portable paths in reports.** All findings use `relative/path/to/file.php:LINE` — never the absolute host path. The frontmatter records `repo` (the folder name). A reader on any host can resolve absolute paths themselves.
- **No secrets in output.** Redact hardcoded credentials to the last 4 characters in every report, including `overview.md`.
- **Large repos.** If the PHP file count exceeds ~2,000, tell the user up front and ask whether to (a) proceed full-scope, (b) scope to the public webroot only, or (c) scope to a subdirectory they name. Record the chosen scope in `overview.md` frontmatter as `scope: <dir>`.
- **Per-agent timeouts.** If an agent has not responded after a reasonable wait, continue without it and list it under "Perspectives skipped (timed out)" in `overview.md`. Do not block the whole review on one stuck agent.
- **Stale findings check.** Before including a finding in `overview.md`, verify the `file:line` still exists and still matches the quoted code at the reviewed commit. Drop findings that don't resolve.
- **No cross-repo writes.** Write only under `<workspace_root>/docs/review-codebase/<repo>/`. Nothing anywhere else.
- **Never bulk-load property Overview docs into context.** If an agent needs orientation, point it at a path — do not inline `Overview.md` into the prompt. Every claim ships against code, not against Overview prose.

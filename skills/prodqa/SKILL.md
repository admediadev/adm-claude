---
name: prodqa
description: Deploy a specific git commit of an AdMedia property to the STAGING (QA) environment using the `prodqa` command. Use when a PR/branch needs to be QA'd on staging before a production release.
---

# prodqa — deploy a commit to staging (QA)

`prodqa` pushes a single git commit of a property to the **staging / QA**
environment so it can be tested against realistic data before a production
release. It does **not** deploy to production — production releases go through
the separate release process (handled by the release team / "released" in CHGREL).

## The command

`/usr/bin/prodqa` is a thin wrapper around the push service:

```sh
#!/bin/sh
curl "http://push-oci.admedia.com/prodqa.php?project=$1&commitHash=$2&user=`whoami`"
```

So it just tells `push-oci.admedia.com` to deploy `commitHash` of `project` to
staging, tagged with the current user.

## Usage

```bash
prodqa <project> <commitHash>
```

- **`<project>`** — the property/repo, named by its hostname directory, e.g.
  `advertisers7_new.admedia.com`, `admediacrons.com`, `mngt.admedia.com`.
- **`<commitHash>`** — the git commit to deploy (full or long-short hash).

### Example

```bash
# Deploy the Stats_model fix commit of advertisers7_new to staging:
prodqa advertisers7_new.admedia.com 9300f699860
```

## Prerequisites

- **The commit must be pushed to the remote (`bit.admedia.com`)** first — the
  push service fetches the commit from the repo, so it has to exist on origin
  (a branch or merged). A purely local commit won't deploy.
- Use the exact commit you want on staging (e.g. the head of the PR branch).
  Deploying the branch-head commit is fine even before the PR is merged — it
  deploys that exact code to staging for QA.

## After running

- It's a fire-and-forget `curl`; check the response body for success/queued
  status. If it errors, confirm the commit is pushed and the project name
  matches the repo hostname exactly.
- Staging then runs that code. For DB-driven features, remember staging reads
  the shared prod databases — so any data the feature needs (e.g. cron output
  tables) must already exist / be regenerated in those DBs.

## Notes

- Staging-only. Do **not** treat a `prodqa` deploy as a production release.
- One commit per property per call. To QA changes spanning multiple repos, run
  `prodqa` once per repo with that repo's commit.

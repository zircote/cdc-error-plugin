---
id: how-to-catalog-pinning
type: procedural
created: 2026-07-03T17:38:07-04:00
diataxis_type: how-to
title: How the marketplace catalog gets pinned to a release
---

# How the marketplace catalog gets pinned to a release

This plugin resides *inside* its own marketplace: `.claude-plugin/marketplace.json`'s
`error-handling` entry points at this same repository. Every release must
re-pin that entry to the released commit's exact SHA — a self-hosted entry
gets the same discipline an external SHA-pinned catalog entry gets. This is
part of the release, not a separate manual step, and this document is the
runbook for both the automated path and the manual fallback.

## What happens automatically

`.github/workflows/release.yml`'s `pin-catalog` job runs after `publish`
succeeds, on every tag push (`v*.*.*`), never on `workflow_dispatch` dry-runs:

1. Checks out `main` using the `CATALOG_PIN_TOKEN` secret (see
   [Why this token exists](#why-this-uses-a-pat-not-the-default-token) below).
2. Compares `marketplace.json`'s current pinned SHA against the just-released
   commit (`github.sha`, the exact commit the tag points to). If they already
   match, it exits without changing anything — this makes the job idempotent
   if it's ever re-run against the same tag.
3. Otherwise: rewrites the `error-handling` plugin entry's `source` to
   `{"source": "github", "repo": "zircote/cdc-error-plugin", "ref": "<tag>",
   "sha": "<release-sha>"}`, and bumps `marketplace.json`'s own `version`
   field by one patch.
4. Commits (as `cdc-error-plugin-catalog-bot`) and pushes directly to `main`.

## Verify it worked

After a release, confirm the catalog was actually re-pinned:

```bash
git fetch origin main
git log origin/main -1 --format='%H %s'
# Expect: chore(catalog): pin error-handling to v<X.Y.Z> (<short-sha>)

jq '.version, .plugins[0].source' <(git show origin/main:.claude-plugin/marketplace.json)
# .source.sha should equal the release tag's commit:
git rev-parse v<X.Y.Z>
```

If the `pin-catalog` job's run shows a failure, check the job logs first —
most failures are either a rejected push (see below) or an expired/revoked
`CATALOG_PIN_TOKEN`.

## Manual fallback (if the automated job fails)

Do this exactly once, by hand, if `pin-catalog` fails and you need the
catalog current before diagnosing the automation:

```bash
TAG="v0.4.2"                                      # the tag you just released
RELEASE_SHA="$(git rev-parse "${TAG}")"
FILE=.claude-plugin/marketplace.json

CURRENT_VERSION="$(jq -r '.version' "${FILE}")"
# Bump the patch component of CURRENT_VERSION yourself, e.g. 1.0.1 -> 1.0.2
NEW_VERSION="1.0.2"

jq --arg repo "zircote/cdc-error-plugin" --arg ref "${TAG}" --arg sha "${RELEASE_SHA}" \
   --arg newver "${NEW_VERSION}" --arg name "error-handling" \
  '.version = $newver
   | .plugins = [.plugins[] | if .name == $name
       then .source = {"source": "github", "repo": $repo, "ref": $ref, "sha": $sha}
       else . end]' \
  "${FILE}" > "${FILE}.tmp" && mv "${FILE}.tmp" "${FILE}"

claude plugin validate . --strict   # confirm it's still valid before committing

git add "${FILE}"
git commit -m "chore(catalog): pin error-handling to ${TAG} (${RELEASE_SHA:0:7})"
git push origin main   # succeeds because you're an actual repo admin
```

This is the exact logic `pin-catalog` runs; the only difference is you're
pushing with your own admin credentials instead of the PAT.

## Why this uses a PAT, not the default token

`main` has branch protection requiring `pin-check`, `actionlint`, and
`claude plugin validate` to pass before a push lands. An authenticated admin
user's push bypasses this (`enforce_admins: false` exempts admins) — that's
why manual pushes throughout this repo's history succeed with a "Bypassed
rule violations" notice. The default `GITHUB_TOKEN` a workflow run gets is
**not** treated as that kind of admin-bypass actor, so a direct push from
`pin-catalog` using the default token would very likely be rejected.

`CATALOG_PIN_TOKEN` is a fine-grained personal access token, scoped to only
this repository, with `Contents: Read and write` permission, stored as a
repository secret. Because it authenticates as the actual repo owner, the
push it makes bypasses the same way a manual admin push does.

**Token rotation**: fine-grained PATs expire. When `CATALOG_PIN_TOKEN`
expires, `pin-catalog` will fail on the next release with a `403`/auth error
from `actions/checkout` or `git push`. Generate a new fine-grained token
(same scope: this repo only, Contents: Read and write) and replace the
secret:

```bash
gh secret set CATALOG_PIN_TOKEN --repo zircote/cdc-error-plugin
```

## Why this only applies to the self-hosted entry

The pin-catalog logic targets the `error-handling` plugin entry specifically
by name (`select(.name == "error-handling")`), not by position. If this
marketplace ever lists additional plugins that are genuinely external
(fetched from a different repo), those entries are pinned by their own
release processes, not this one — `pin-catalog` only re-pins the plugin that
lives inside this same repo.

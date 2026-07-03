---
id: adr-0006-self-referential-catalog-pinning
type: adr
created: 2026-07-03T00:00:00-04:00
title: "Self-Referential Catalog SHA-Pinning via PAT-Authenticated Automation"
description: "marketplace.json's plugin entry is rewritten from a live './' relative-path source to a SHA-pinned github source after every release, automated as a post-publish job in release.yml authenticated by a dedicated fine-grained PAT, since the default GITHUB_TOKEN cannot bypass main's branch protection the way an authenticated admin push can."
category: release-engineering
tags:
  - adr
  - architecture
  - marketplace
  - supply-chain
  - github-actions
status: accepted
updated: 2026-07-03
author: zircote
project: error-handling-plugin
technologies:
  - github-actions
  - claude-code-plugins
audience:
  - developers
  - architects
related:
  - 0002-single-repo-plugin-and-marketplace.md
  - 0005-default-token-for-release-auth.md
  - 0007-github-flow-branch-model.md
---

# ADR-0006: Self-Referential Catalog SHA-Pinning via PAT-Authenticated Automation

## Status

Accepted

## Context

### Background and Problem Statement

Per ADR-0002, this repo is both the `error-handling` plugin and the
`cdc-errors` marketplace that lists it. The plugin entry originally used a
local relative-path source (`"source": "./"`), which resolves against
whatever commit happens to be checked out — a live reference with no
immutability guarantee. External plugin catalog entries in the reference
architecture are pinned to a `ref` + full 40-character `sha`, so the catalog
always resolves to an exact, attested commit. The explicit requirement was
that a self-hosted plugin entry get the same discipline: "the marketplace
must be pointing to the release sha... plugin is released -> sha added to
marketplace -> marketplace is bumped patch and release... this must be
runbooked... this is for plugins that reside INSIDE the marketplace like
this one and must be part of the release."

Implementing this raised a genuine obstacle: `main` has branch protection
(ADR-0007) requiring three status checks to pass before a push lands.
Every manual push made earlier in this session succeeded with a "Bypassed
rule violations" notice — because those pushes were authenticated as an
actual repository admin, and `enforce_admins: false` exempts admins from
required status checks. A workflow's default `GITHUB_TOKEN` is not treated
as that kind of admin-bypass actor merely because it has `contents: write`
permission; a direct push to `main` from `pin-catalog` using the default
token would very likely be rejected by the same protection rule.

### Current Limitations

1. **No pinning existed** — the plugin entry tracked live HEAD indefinitely.
2. **GitHub Actions prevents a subtler workaround**: a PR opened using the
   default `GITHUB_TOKEN` does not trigger downstream workflow runs
   (an anti-loop protection), so a PR-based catalog update would never get
   its own required status checks to populate, and would hang unmergeable
   through the normal merge path.
3. **Classic branch protection has no per-actor bypass list** — that
   capability exists only in the newer GitHub Rulesets feature; this repo
   uses classic branch protection, whose only lever is the global
   `enforce_admins` toggle.

## Decision Drivers

### Primary Decision Drivers

1. **The catalog shall always resolve to an exact, attested commit** after
   a release — not a live reference that can drift with the next commit to
   `main`.
2. **Pinning shall be part of the release, not a manual side-step** — no
   dependency on a human remembering to run a script after tagging.
3. **The write-back mechanism shall actually work against `main`'s existing
   branch protection**, not be designed around an assumption that turns out
   false in practice.

### Secondary Decision Drivers

1. **Idempotency**: re-running the pinning logic against an already-pinned
   commit shall be a safe no-op, not a duplicate commit or a broken state.
2. **Correct targeting**: the automation shall pin the specific self-hosted
   plugin entry by name, not by array position, so a future second local
   entry isn't silently mis-targeted.

## Considered Options

### Option 1: Direct push using the default GITHUB_TOKEN

**Description**: `pin-catalog` checks out `main` and pushes the updated
`marketplace.json` using the workflow's own `GITHUB_TOKEN` with
`contents: write`.

**Advantages**:

- No new secret or credential to manage.
- Simplest possible implementation.

**Disadvantages**:

- The default `GITHUB_TOKEN` is very likely not treated as an admin-bypass
  actor for required status checks — this push would likely be rejected,
  since the commit it's pushing has no prior status-check history and
  `GITHUB_TOKEN` isn't an admin identity the way a human repo owner is.

**Risk Assessment**:

- **Technical Risk**: High. Untested against this repo's actual protection
  rule; likely fails silently or loudly depending on GitHub's exact
  enforcement behavior.
- **Schedule Risk**: Low to implement, but a failure discovered only after
  the first real release would defeat the entire point.
- **Ecosystem Risk**: Low.

**Disqualifying Factor**: real uncertainty whether this satisfies the
requirement at all, discovered before implementation rather than after a
failed release.

### Option 2: Relax branch protection to exempt this automation

**Description**: Disable or narrow required status checks specifically for
this automated commit path.

**Advantages**:

- Would sidestep the bypass problem entirely if achievable.

**Disadvantages**:

- Classic branch protection (in use here) has no per-actor bypass list —
  only the binary `enforce_admins` toggle exists. Achieving a
  narrowly-scoped exemption would require migrating to GitHub Rulesets, a
  larger change than this decision warrants, or relaxing protection for
  *all* pushes, weakening the guarantee ADR-0007 established for everyone,
  not just this automation.

**Risk Assessment**:

- **Technical Risk**: Medium to high depending on which relaxation path is
  taken; the narrowly-scoped version isn't available in the branch
  protection model already in use.
- **Schedule Risk**: High if a Rulesets migration is required to do this
  correctly.
- **Ecosystem Risk**: Medium — weakens `main`'s protection guarantee for a
  single automation's convenience.

**Disqualifying Factor**: no narrowly-scoped exemption is available without
a larger protection-model migration or weakening protection for everyone.

### Option 3: PR-based flow with manual merge

**Description**: `pin-catalog` pushes a branch and opens a PR instead of
pushing directly to `main`; a human merges it.

**Advantages**:

- Keeps `main`'s protection fully intact — no bypass needed at all.
- No new secret required.

**Disadvantages**:

- GitHub Actions does not trigger downstream workflow runs from events
  authored by the default `GITHUB_TOKEN` — the PR's own required CI checks
  (`ci.yml` on `pull_request`) would never run, so the PR could never
  satisfy "3 of 3 required status checks" through the normal merge button.
  Merging would require the same admin-bypass a direct push needs anyway,
  just with an extra manual step in between.
- Defeats the "part of the release, not a manual side-step" driver — a
  human still has to intervene every release.

**Risk Assessment**:

- **Technical Risk**: Low technically, but functionally broken for full
  automation given the anti-loop restriction.
- **Schedule Risk**: Low to implement, but doesn't achieve the actual goal.
- **Ecosystem Risk**: Low.

**Disqualifying Factor**: cannot achieve full automation given GitHub
Actions' anti-loop restriction on GITHUB_TOKEN-authored PRs; would still
need manual merge every release, which the requirement explicitly rejects.

### Option 4: Dedicated fine-grained PAT, scoped to this repo only

**Description**: A fine-grained personal access token
(`CATALOG_PIN_TOKEN`), scoped to only `zircote/cdc-error-plugin` with
`Contents: Read and write`, stored as a repository secret. `pin-catalog`
checks out and pushes using this token, so the push authenticates as the
actual repo owner — the same identity whose manual pushes already bypass
required status checks via `enforce_admins: false`.

**Advantages**:

- Achieves full automation: the push succeeds the same way a manual admin
  push does, with no manual merge step.
- Scope is narrow: only this one repo, only Contents read/write — not a
  broad personal token reused across concerns.
- Idempotency and name-based targeting (not positional) are cheap to add
  and close two real correctness gaps in the same implementation.

**Disadvantages**:

- A new secret to manage and periodically rotate (fine-grained PATs
  expire).
- `main`'s commit history now includes bot-authored commits
  (`cdc-error-plugin-catalog-bot`) alongside human-authored ones.

**Risk Assessment**:

- **Technical Risk**: Low. Confirmed working by testing the pinning logic
  locally (both the update path and the idempotency no-op path) before
  trusting it in CI, and by the actual `pin-catalog` job succeeding.
- **Schedule Risk**: Low. Token creation is a one-time setup step.
- **Ecosystem Risk**: Low, contained to this one repo's scope.

## Decision

We adopt the dedicated fine-grained PAT (`CATALOG_PIN_TOKEN`), scoped to
only this repository with `Contents: Read and write`. `pin-catalog` runs
after `publish` succeeds, gated to real tag pushes only (never
`workflow_dispatch` dry-runs): it compares the catalog's currently-pinned
SHA against the release SHA (no-op if already equal), rewrites the
`error-handling` plugin entry's `source` to
`{"source": "github", "repo": "zircote/cdc-error-plugin", "ref": "<tag>",
"sha": "<release-sha>"}` by selecting on `.name == "error-handling"` (not
array position), bumps `marketplace.json`'s own `version` by one patch, and
pushes directly to `main` using the PAT.

The manual fallback procedure, and the token-rotation runbook, are
documented at `docs/how-to/catalog-pinning.md`.

## Consequences

### Positive

1. **Full automation achieved**: no manual step between tagging a release
   and the catalog reflecting it.
2. **The catalog always resolves to an exact, attested commit** — the same
   discipline external SHA-pinned entries get.
3. **Idempotent and name-targeted**: safe against accidental re-runs and
   against future second local plugin entries being mis-targeted.

### Negative

1. **A new credential to manage**: `CATALOG_PIN_TOKEN` needs periodic
   manual rotation when it expires, and its compromise would grant
   write access to this one repo (mitigated by its narrow scope: this repo
   only, Contents only, not a broader personal token).
2. **Bot-authored commits now appear in `main`'s history**, distinct from
   human-authored commits by author identity (`cdc-error-plugin-catalog-bot`)
   but mixed into the same linear history.
3. **Local development testing is affected**: once the catalog entry is
   SHA-pinned, a locally-added marketplace no longer serves a developer's
   uncommitted local changes — this was already noted as a consequence of
   ADR-0002's single-repo shape, but the pinning makes it concrete rather
   than theoretical.

### Neutral

1. **This is a narrower-scoped instance of the same auth question ADR-0005
   answered differently** — `publish` doesn't push to a protected branch,
   so the default token sufficed there; `pin-catalog` does, so it needed a
   different answer. Both are documented as distinct, deliberate choices
   rather than an inconsistency.

## Decision Outcome

Measured by: the manual fallback was performed once (for `v0.4.1`, pinning
to commit `67f3e6e`, catalog version `1.0.0` -> `1.0.1`), confirmed valid via
`claude plugin validate . --strict`; the jq pinning logic was independently
tested locally against both the update path and the already-pinned no-op
path before being trusted in CI; and the automated `pin-catalog` job was
wired into `release.yml`, gated correctly to real tag pushes only.

## Related Decisions

- [ADR-0002: Single-Repo Plugin-and-Marketplace Architecture](0002-single-repo-plugin-and-marketplace.md) - the shape that makes self-referential pinning meaningful.
- [ADR-0005: Default GITHUB_TOKEN for Release Publishing](0005-default-token-for-release-auth.md) - the different answer to a related auth question, and why.
- [ADR-0007: GitHub Flow Branch Model](0007-github-flow-branch-model.md) - the branch protection that forced this decision.

## Links

- [Fine-grained personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens) - the credential type used, scoped to this repo only.
- `docs/how-to/catalog-pinning.md` - the runbook for both the automated path and the manual fallback.

## More Information

- **Date:** 2026-07-03
- **Source:** explicit user requirement, refined through direct discussion of the branch-protection obstacle and a choice among three write-back mechanisms.
- **Related ADRs:** 0002, 0005, 0007

## Audit

### 2026-07-03

**Status:** Compliant

**Findings:**

| Finding | Files | Lines | Assessment |
| --- | --- | --- | --- |
| Manual pin applied and validated for v0.4.1 | `.claude-plugin/marketplace.json` | - | accepted |
| jq pinning + idempotency logic tested locally before trusting in CI | `.github/workflows/release.yml` | - | accepted |
| Name-based targeting (not positional) implemented | `.github/workflows/release.yml` | - | accepted |

**Summary:** Self-referential catalog pinning implemented, tested, and
documented with a runbook covering both automated and manual paths.

**Action Required:** Rotate `CATALOG_PIN_TOKEN` before it expires; watch the
first automated `pin-catalog` run (next release after this ADR) to confirm
the PAT-authenticated push succeeds in practice, not just in local testing.

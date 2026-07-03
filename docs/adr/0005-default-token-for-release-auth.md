---
id: adr-0005-default-token-for-release-auth
type: adr
created: 2026-07-03T00:00:00-04:00
title: "Default GITHUB_TOKEN for Release Publishing"
description: "The release pipeline's publish job (creating the GitHub Release) uses the workflow's own default GITHUB_TOKEN, scoped per job, rather than minting tokens from dedicated GitHub Apps the way the multi-repo reference architecture does for its release/catalog/pages/automerge/ci identities."
category: release-engineering
tags:
  - adr
  - architecture
  - authentication
  - github-actions
  - release
status: accepted
updated: 2026-07-03
author: zircote
project: error-handling-plugin
technologies:
  - github-actions
audience:
  - developers
  - architects
related:
  - 0006-self-referential-catalog-pinning.md
---

# ADR-0005: Default GITHUB_TOKEN for Release Publishing

## Status

Accepted

## Context

### Background and Problem Statement

The reference architecture studied for this work maintains five
least-privilege GitHub Apps at the org level (`ci`, `catalog`, `pages`,
`automerge`, `release`), each minting short-lived tokens for a specific
purpose — the `release` App specifically authenticates `gh release
create`/`gh release upload`, kept deliberately separate from the OIDC
identity that produces attestations. This exists to give each automated
concern its own least-privilege, revocable identity across a multi-repo org
with several collaborators.

This repo is solo-maintained, with no other collaborators and no fleet of
sibling repos those Apps would serve. The question is whether `publish`
(the job that actually creates the GitHub Release and uploads its assets)
needs a dedicated App identity, or whether the default `GITHUB_TOKEN` —
scoped to `contents: write` for that job only — is sufficient.

### Current Limitations

1. **No existing release-publish mechanism** prior to this decision.
2. **App registration has real setup cost**: creating a GitHub App, storing
   its private key and client ID as org variables/secrets, and wiring
   `actions/create-github-app-token` is meaningful ceremony for a single
   repo with a single publish concern.

## Decision Drivers

### Primary Decision Drivers

1. **Least privilege, not zero privilege**: the publish job's token shall
   have exactly the permission it needs (`contents: write`, scoped to that
   job) and no more — a bar the default `GITHUB_TOKEN` already clears when
   scoped per-job, without requiring a separate App.
2. **No unshared App infrastructure**: an App registered solely to serve
   this one repo's one publish job doesn't provide isolation benefit beyond
   what job-scoped `permissions:` already gives, since there's no second
   repo or collaborator the App's revocability protects against.

### Secondary Decision Drivers

1. **Setup cost proportional to actual need**: avoid GitHub App
   registration, private key storage, and org-variable wiring when the
   default token already satisfies the permission model.

## Considered Options

### Option 1: Dedicated GitHub App for release publishing

**Description**: Register a `release` GitHub App (or reuse a pattern like
the reference org's), mint a token via
`actions/create-github-app-token`, and use it for `gh release create`.

**Advantages**:

- A compromised default `GITHUB_TOKEN` (scoped to this run) and a
  compromised release-publish identity would be two different blast radii
  to reason about.
- Matches the reference architecture's least-privilege identity separation
  exactly.

**Disadvantages**:

- App registration, private-key storage, and org-variable/secret wiring for
  a single repo with a single collaborator (the repo owner) and one publish
  concern.
- No second repo or collaborator for the isolation to protect against in
  practice.

**Risk Assessment**:

- **Technical Risk**: Low; proven pattern.
- **Schedule Risk**: Medium. Real setup ceremony before the first release
  could ship.
- **Ecosystem Risk**: Low.

**Disqualifying Factor**: disproportionate setup cost for a solo repo with
no second identity or collaborator to isolate against.

### Option 2: Default GITHUB_TOKEN, scoped per job

**Description**: The `publish` job's `permissions:` block grants only
`contents: write`; every other job (`build`, `gate-*`, `sbom`, `sign-catalog`,
`vex`, `verify`) gets only the permissions it actually uses
(`id-token: write`/`attestations: write`/`security-events: write` as
applicable, never `contents: write` unless it genuinely writes).

**Advantages**:

- No App registration, no private key to store or rotate.
- Least privilege is still achieved via per-job `permissions:` scoping — the
  same principle the App-based approach serves, applied at the workflow
  level instead of the identity level.

**Disadvantages**:

- No separate revocable identity if the default token needs to be treated as
  compromised — the mitigation is standard GitHub Actions token hygiene
  (short-lived, run-scoped tokens), not a distinct App to revoke.

**Risk Assessment**:

- **Technical Risk**: Low. Per-job permission scoping already verified
  correct by an independent review (`arbiter:impartial-reviewer`) during
  implementation — no job holds broader permission than its steps use.
- **Schedule Risk**: Low. No setup ceremony before the first release.
- **Ecosystem Risk**: Low.

## Decision

We adopt the default `GITHUB_TOKEN`, scoped per job via each job's own
`permissions:` block, for release publishing and every other job in
`release.yml`. No GitHub Apps are registered for this repo's release
pipeline. Note this is a distinct decision from ADR-0006's catalog-pinning
write-back, which *does* use a dedicated credential (a fine-grained PAT) —
that decision was forced by branch protection requiring an actual
admin-bypass identity, a constraint that doesn't apply to `publish` (which
creates a Release, not a protected-branch push).

## Consequences

### Positive

1. **No App registration, private-key storage, or rotation burden** for
   release publishing specifically.
2. **Least privilege still achieved** via per-job permission scoping,
   independently verified.

### Negative

1. **No separate revocable identity** for the publish concern — if the
   default token needs to be treated as compromised, the blast radius is
   "this run," not narrower.

### Neutral

1. **This is a per-concern decision, not a blanket policy**: ADR-0006's
   catalog-pinning write-back needed a different answer (a PAT) because it
   pushes to a *protected branch*, a constraint `publish` doesn't share.

## Decision Outcome

Measured by: the `v0.4.1` GitHub Release was created successfully using only
the default `GITHUB_TOKEN`, scoped to `contents: write` in the `publish`
job's `permissions:` block, with every other job's permissions independently
reviewed and confirmed least-privilege.

## Related Decisions

- [ADR-0006: Self-Referential Catalog SHA-Pinning via PAT-Authenticated Automation](0006-self-referential-catalog-pinning.md) - the *different* auth decision made for the catalog-pinning write-back, and why it needed a PAT where `publish` doesn't.

## Links

- [GitHub Actions: Automatic token authentication](https://docs.github.com/en/actions/security-guides/automatic-token-authentication) - the default `GITHUB_TOKEN` this decision relies on.

## More Information

- **Date:** 2026-07-03
- **Source:** explicit user choice between "default GITHUB_TOKEN" and "dedicated GitHub App(s)" when attestation-depth scope was decided.
- **Related ADRs:** 0006

## Audit

### 2026-07-03

**Status:** Compliant

**Findings:**

| Finding | Files | Lines | Assessment |
| --- | --- | --- | --- |
| Per-job permissions independently reviewed, no over-broad grants found | `.github/workflows/release.yml` | - | accepted |

**Summary:** Default-token approach implemented and verified least-privilege
by an independent review pass before the first release.

**Action Required:** None. Revisit only if this repo gains collaborators or
sibling repos that would benefit from a separate revocable release identity.


---
id: adr-0004-no-central-attestation-signer
type: adr
created: 2026-07-03T00:00:00-04:00
title: "Self-Signed Attestations, No Central Signer"
description: "Every attestation and the marketplace catalog's cosign signature are produced by this repo's own release.yml workflow, rather than a separate central-repo signer the way the multi-repo reference architecture does it, since this repo has no fleet of sibling repos to share a signer with."
category: release-engineering
tags:
  - adr
  - architecture
  - attestation
  - supply-chain
  - slsa
status: accepted
updated: 2026-07-03
author: zircote
project: error-handling-plugin
technologies:
  - slsa
  - sigstore
  - cosign
  - github-actions
audience:
  - developers
  - architects
related:
  - 0003-full-tier-attestation-scope.md
---

# ADR-0004: Self-Signed Attestations, No Central Signer

## Status

Accepted

## Context

### Background and Problem Statement

Under SLSA Build L3, the strongest guarantee comes from separating the
*artifact repo* from the *signer identity*: a central, shared reusable
workflow (in its own repo) performs the actual signing, so the Fulcio
certificate's SAN names the central signer, not whatever repo is being
released. The reference architecture studied for this work does exactly
this — `modeled-information-format/.github`'s `reusable-attest-scan.yml`
and `reusable-cosign-sign.yml` sign on behalf of every caller repo in that
org, and consumer verification pins `--signer-workflow` to that one shared
identity across all of the org's repos.

This repo has no sibling repos to share a signer with — it is the only
artifact this maintainer publishes through this exact pipeline. Building a
separate central-signer repo for an audience of one adds a second thing to
keep secure (its own permissions, its own pin-checked workflows) without a
second artifact repo to actually benefit from sharing it.

### Current Limitations

1. **No existing signer infrastructure** — before this decision, this repo
   had no attestation pipeline at all.
2. **A central signer's value is proportional to the number of callers** —
   with exactly one caller, the isolation benefit (a compromised artifact
   repo can't forge its own attestations) doesn't apply in the same way,
   since the artifact repo and the only possible "org" are the same thing.

## Decision Drivers

### Primary Decision Drivers

1. **Proportionality to actual repo count**: the signer topology shall match
   the number of repos actually being signed for (one), not a multi-repo
   pattern with no second repo to share it with.
2. **No unshared infrastructure**: a signer repo that exists solely to sign
   for its own creator's other single repo adds operational surface without
   the isolation benefit a shared signer provides across genuinely
   independent callers.

### Secondary Decision Drivers

1. **Verification must still pin the signer explicitly**: even without a
   separate repo, verification shall not rely on `--repo` alone (which only
   proves "signed by some workflow in this repo," not specifically the
   release workflow) — `--signer-workflow` still needs to name the exact
   workflow.

## Considered Options

### Option 1: Central signer repo (mirrors the reference org)

**Description**: Create a second repository hosting reusable
`attest-scan`/`cosign-sign` workflows; `release.yml` in this repo calls them
via `uses: <central-repo>/.github/workflows/reusable-attest-scan.yml@<sha>`.

**Advantages**:

- Matches SLSA Build L3's strongest isolation guarantee: a compromised
  artifact repo alone can't forge attestations, since signing happens in a
  separate repo's identity.
- Scales cleanly if this maintainer ever publishes a second plugin/artifact
  through the same signer.

**Disadvantages**:

- A second repo to secure, pin-check, and maintain, for an audience of
  exactly one caller today.
- Verification commands would need `--signer-workflow` pinned to a repo
  distinct from the artifact repo — more moving parts to document and keep
  correct (this repo's own docs already needed correcting once this
  session, when `--signer-workflow` was initially omitted from verification
  commands entirely; a second repo in the identity would add another axis
  to get wrong).

**Risk Assessment**:

- **Technical Risk**: Low; proven pattern.
- **Schedule Risk**: Medium. A second repo's CI, branch protection, and pin
  discipline all need setting up before the first repo can even use it.
- **Ecosystem Risk**: Low.

**Disqualifying Factor**: no second artifact repo exists to justify the
isolation benefit's cost.

### Option 2: Self-signed, this repo's own workflow is the signer

**Description**: `release.yml` in this same repo runs `actions/attest`,
`actions/attest-build-provenance`, `actions/attest-sbom`, and
`sigstore/cosign-installer` directly. Verification pins
`--signer-workflow zircote/cdc-error-plugin/.github/workflows/release.yml`
and, for the catalog signature, a `--certificate-identity-regexp` anchored
to that same workflow path.

**Advantages**:

- No second repo to secure or maintain.
- Verification is uniform: every `gh attestation verify` command in
  `SECURITY.md`/`docs/how-to/verify-release.md` names the same repo and
  workflow — one identity to get right, not two.

**Disadvantages**:

- The isolation guarantee is weaker than a true central signer: a
  compromise of this one repo's Actions configuration compromises both the
  artifact *and* its own attestations, since they're produced by the same
  identity. (Mitigated by: the fail-closed `verify` job still requires
  every attestation to actually verify before `publish` runs, and required
  status checks + no direct-push-without-bypass protect `main` itself.)

**Risk Assessment**:

- **Technical Risk**: Low for a single-repo scope; the residual risk (repo
  compromise compromises both artifact and signer) is explicitly named, not
  hidden.
- **Schedule Risk**: Low. No second repo to stand up first.
- **Ecosystem Risk**: Low.

## Decision

We adopt self-signed attestations: `release.yml` running in this same repo
is the signer for every attestation (provenance, SBOM, all six gate
predicates, VEX) and for the cosign catalog signature. Verification pins
`--signer-workflow` (for `gh attestation verify`) and
`--certificate-identity-regexp` (for `cosign verify-blob`) to this repo's
own `release.yml`, never relying on `--repo` alone.

## Consequences

### Positive

1. **No second repo to secure, pin-check, or keep current** for a
   single-artifact publisher.
2. **Verification is uniform** — one repo, one workflow path, named
   consistently across every documented verification command.

### Negative

1. **Weaker isolation than a true central signer**: a compromise of this
   repo's own Actions configuration could in principle forge attestations
   for its own artifact, since artifact and signer share an identity. This
   residual risk is accepted and named explicitly (see
   `docs/explanation/attested-releases.md`), not hidden behind an
   attestation that implies stronger guarantees than it has.

### Neutral

1. **If this maintainer later publishes a second artifact** through a
   shared pipeline, the central-signer option becomes worth revisiting —
   this ADR's decision is scoped to "one artifact repo today," not a
   permanent rejection of the pattern.

## Decision Outcome

Measured by: every attestation on the published `v0.4.1` release
independently re-verified with `--signer-workflow` pinned to
`zircote/cdc-error-plugin/.github/workflows/release.yml`, and the cosign
catalog signature independently re-verified with
`--certificate-identity-regexp` anchored to the same workflow path — both
confirmed from a clean directory after the release, per the documented
commands in `docs/how-to/verify-release.md`.

## Related Decisions

- [ADR-0003: Full-Tier Attestation Scope for the Release Pipeline](0003-full-tier-attestation-scope.md) - what gets signed by this signer.

## Links

- [SLSA Build Levels](https://slsa.dev/spec/v1.0/levels) - the L3 central-signer guarantee this decision explicitly trades away for a single-repo scope.
- [gh attestation verify documentation](https://cli.github.com/manual/gh_attestation_verify) - `--signer-workflow` is the flag this decision's verification model depends on.

## More Information

- **Date:** 2026-07-03
- **Source:** architectural analysis during full-tier attestation implementation; corrected mid-session after an independent review found `--signer-workflow` had been omitted from the verify job's own commands, weakening the guarantee to "signed by some workflow in this repo."
- **Related ADRs:** 0003

## Audit

### 2026-07-03

**Status:** Compliant

**Findings:**

| Finding | Files | Lines | Assessment |
| --- | --- | --- | --- |
| `--signer-workflow` initially missing from verify job and docs | `.github/workflows/release.yml`, `SECURITY.md`, `docs/how-to/verify-release.md` | - | fixed, then accepted |

**Summary:** An independent review caught that the fail-closed `verify` job
omitted `--signer-workflow`, weakening it to `--repo`-only checking. Fixed
across the workflow and every consumer-facing verification command before
the `v0.4.1` release shipped.

**Action Required:** None. Revisit if a second artifact repo under this
maintainer's control ever needs to share a signer.

---
id: adr-0003-full-tier-attestation-scope
type: adr
created: 2026-07-03T00:00:00-04:00
title: "Full-Tier Attestation Scope for the Release Pipeline"
description: "The release pipeline attests SLSA build provenance, a CycloneDX SBOM, six quality-gate verdicts (SAST/SCA/Trivy/Semgrep/secrets/manifest), and an OpenVEX disposition, and cosign-signs the marketplace catalog, rather than a lighter provenance-only tier, despite this plugin having no runtime dependency tree."
category: release-engineering
tags:
  - adr
  - architecture
  - attestation
  - supply-chain
  - release
status: accepted
updated: 2026-07-03
author: zircote
project: error-handling-plugin
technologies:
  - slsa
  - cyclonedx
  - openvex
  - cosign
  - github-actions
audience:
  - developers
  - architects
related:
  - 0004-no-central-attestation-signer.md
---

# ADR-0003: Full-Tier Attestation Scope for the Release Pipeline

## Status

Accepted

## Context

### Background and Problem Statement

Attested delivery (SLSA provenance, SBOMs, signed gate verdicts) is
established practice for supply-chain-conscious software, exemplified by the
reference architecture studied for this work
([modeled-information-format/claude-code-plugins](https://github.com/modeled-information-format/claude-code-plugins)),
which runs SAST, SCA, IaC/license scanning, Semgrep, secrets scanning,
manifest review, SBOM generation, OpenVEX disposition, and cosign catalog
signing on every release, each seam-attested and fail-closed verified before
publish. This plugin has no runtime dependency tree — it is a markdown/JSON
skill bundle — which raises the question of whether that machinery is
proportionate here, or whether a lighter provenance-only tier is the honest
choice for what this artifact actually is.

### Current Limitations

1. **No prior release process existed** — the first version of
   `release.yml` built a tarball with no attestation at all.
2. **"No dependencies" is a claim, not a permanent fact** — nothing enforced
   that claim or would catch it becoming false (e.g. if a future eval
   harness gains its own `package.json`).
3. **The repo's actual attack surface is its own CI/CD logic**, not
   application code — a bug in `release.yml` that weakens fail-closed
   verification is a real, live risk class this plugin does have, even
   without a dependency tree.

## Decision Drivers

### Primary Decision Drivers

1. **Proportionality**: the pipeline's scope shall match what this artifact
   actually is (a skill bundle with a CI/CD surface, not a compiled binary
   with a dependency graph) without being disproportionate in either
   direction.
2. **Forward-looking coverage**: gates that report "nothing found" against
   an empty dependency graph today shall still be wired, so a future
   dependency doesn't silently escape scanning.
3. **No theater**: a gate with genuinely nothing to scan shall be omitted
   rather than wired for the appearance of coverage.

### Secondary Decision Drivers

1. **Consumer trust parity**: a consumer who already verifies attestations
   from other supply-chain-conscious software should be able to point the
   same tooling (`gh attestation verify`, `cosign verify-blob`) at this
   plugin and get a real answer, without learning a bespoke scheme.

## Considered Options

### Option 1: Minimal — provenance + fail-closed verify only

**Description**: `actions/attest-build-provenance` on the release tarball,
re-verified in-run before publish. No SBOM, no per-gate attestations, no
cosign, no VEX.

**Advantages**:

- Matches the `attested-delivery` skill's own guidance for "binaries/bundles
  only" artifacts.
- Minimal CI runtime and maintenance surface.

**Disadvantages**:

- No coverage of the CI/CD surface itself (SAST over the workflow YAML), no
  forward-looking SCA/SBOM if a dependency is later added, no catalog
  signature.

**Risk Assessment**:

- **Technical Risk**: Low.
- **Schedule Risk**: Low. Fastest to implement.
- **Ecosystem Risk**: Low, but consumer trust parity with more mature
  supply-chain practice is weaker.

### Option 2: Medium — provenance + verify + cosign-signed catalog

**Description**: Minimal tier, plus keyless cosign signing of
`marketplace.json` (no SBOM, no per-gate attestations, no VEX).

**Advantages**:

- Adds catalog integrity (a consumer can prove the catalog they fetched is
  the one this repo published) at low incremental cost — no GitHub Apps
  needed, just OIDC.

**Disadvantages**:

- Still no SAST coverage of the release pipeline's own logic, no SBOM/VEX.

**Risk Assessment**:

- **Technical Risk**: Low.
- **Schedule Risk**: Low.
- **Ecosystem Risk**: Low.

### Option 3: Full — mirror the reference architecture

**Description**: SLSA provenance + CycloneDX SBOM (`anchore/sbom-action` +
`actions/attest-sbom`) + six quality-gate verdicts each seam-attested as a
signed custom predicate (SAST via CodeQL `languages: actions`, SCA via
OSV-Scanner, IaC/license via Trivy, Semgrep, secrets via Gitleaks +
TruffleHog, manifest structural review) + OpenVEX disposition + cosign
catalog signature. ShellCheck explicitly omitted: this repo has no `.sh`
scripts, and a gate with nothing to scan produces a permanently-empty
attestation that asserts nothing real.

**Advantages**:

- SAST over the workflow YAML directly covers this repo's actual attack
  surface (its own CI/CD logic).
- SBOM/SCA are forward-looking: if a dependency is ever added, the gate is
  already wired rather than needing retrofitting under time pressure.
- Full parity with the reference architecture's consumer verification
  experience.

**Disadvantages**:

- Eighteen CI jobs per release versus four; higher CI runtime and a larger
  surface of external actions to keep pinned and current.
- Genuinely disproportionate machinery for a repo with no dependency tree,
  by the `attested-delivery` skill's own stated guidance for
  "binaries/bundles only" artifacts.

**Risk Assessment**:

- **Technical Risk**: Low. Every action pin independently verified against
  live GitHub API during implementation (one reference-repo pin,
  `osv-scanner-action`, was found stale and corrected to the tag's actual
  current commit).
- **Schedule Risk**: Medium. Largest implementation effort of the three
  options; caught one real bug (`gate-sast` double-uploading to code
  scanning, since `codeql-action/analyze` already self-uploads) via a
  `workflow_dispatch` dry-run before the first real tag.
- **Ecosystem Risk**: Low.

## Decision

We adopt the **full tier**, explicitly chosen over the proportionate-by-default
minimal tier, on direct instruction: full parity with the reference
architecture was preferred to a scope matched to this artifact's actual
dependency-free nature. ShellCheck is the one gate omitted from the
reference's set, since this repo genuinely has nothing for it to scan.

## Consequences

### Positive

1. **This repo's actual attack surface (its own release pipeline) is
   covered** by SAST, not just its absent dependency tree.
2. **Forward-looking**: SCA/SBOM already exist for the day a dependency is
   added.
3. **Consumer verification parity** with more mature supply-chain practice.

### Negative

1. **Eighteen jobs per release** is real CI runtime and maintenance surface
   for a solo maintainer to keep current (action pins, tool versions like
   Semgrep/Gitleaks/TruffleHog/vexctl).
2. **Disproportionate by the `attested-delivery` skill's own stated
   guidance** for this artifact class — accepted as a deliberate choice, not
   an oversight; see Option 3's disadvantages.
3. **SCA/SBOM currently attest an empty dependency graph** — real, but
   provides less immediate value than it will once a dependency exists.

### Neutral

1. **A future contributor unfamiliar with this decision might reasonably
   ask "why so much machinery for a markdown bundle?"** — this ADR is that
   answer.

## Decision Outcome

Measured by: the full pipeline ran end-to-end for the `v0.4.1` release
(all 18 jobs green, including the fail-closed `verify` job), and every
attestation was independently re-verified from a clean directory afterward
(checksums, provenance, SBOM, all six gate predicates, VEX, and the cosign
catalog signature all passed via the documented `gh attestation verify` /
`cosign verify-blob` commands).

Mitigation for the "disproportionate machinery" negative: `docs/explanation/attested-releases.md`
documents the "why," so the scope reads as a deliberate choice to anyone who
encounters it, not an unexplained accumulation of CI jobs.

## Related Decisions

- [ADR-0004: Self-Signed Attestations, No Central Signer](0004-no-central-attestation-signer.md) - the signer topology for these attestations.
- [ADR-0005: Default GITHUB_TOKEN for Release Publishing](0005-default-token-for-release-auth.md) - the auth model the reference architecture pairs with a central signer, declined here.

## Links

- [SLSA Build Provenance v1](https://slsa.dev/provenance/v1)
- [CycloneDX](https://cyclonedx.org/)
- [OpenVEX](https://openvex.dev/)
- [modeled-information-format/claude-code-plugins](https://github.com/modeled-information-format/claude-code-plugins) - the reference architecture this scope mirrors.

## More Information

- **Date:** 2026-07-03
- **Source:** explicit user choice among three presented tiers (minimal/medium/full), selecting full.
- **Related ADRs:** 0004, 0005

## Audit

### 2026-07-03

**Status:** Compliant

**Findings:**

| Finding | Files | Lines | Assessment |
| --- | --- | --- | --- |
| All 18 jobs green on v0.4.1; all attestations independently re-verified | `.github/workflows/release.yml` | - | accepted |
| Reference pin found stale (osv-scanner-action) and corrected before use | `.github/workflows/release.yml` | - | accepted |
| gate-sast double-upload bug caught by dry-run before first tag | `.github/workflows/release.yml` | - | accepted |

**Summary:** Full-tier attestation implemented, dry-run caught one real bug
before the first real release, and the published `v0.4.1` release verified
clean end-to-end.

**Action Required:** None. Revisit if CI runtime or action-pin maintenance
burden becomes disproportionate to the value delivered.

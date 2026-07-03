---
id: how-to-verify-a-release
type: procedural
created: 2026-07-03T10:00:27-04:00
diataxis_type: how-to
title: Verify a release
---

# Verify a release

Every tagged release publishes a `git archive` tarball carrying SLSA
build-provenance, a CycloneDX SBOM, six quality-gate verdicts, and an OpenVEX
disposition attestation; the `marketplace.json` catalog is cosign-signed. This
walks through re-verifying all of it from a clean workstation. Unlike a
multi-repo org, this plugin has no separate central signer: this repo's own
`release.yml` produces every attestation and the catalog signature. Pin
`--signer-workflow` to that workflow explicitly in every command below —
`--repo` alone only proves "some workflow in this repo signed it," not
specifically the release workflow.

## Prerequisites

- [GitHub CLI](https://cli.github.com/) `gh` ≥ 2.49.0, authenticated:

  ```bash
  gh auth login
  gh auth status
  ```

- [`cosign`](https://github.com/sigstore/cosign), for the catalog signature.

Substitute the actual release tag you're verifying for `TAG` below — the
commands are written against a variable, not a hard-coded version, since a
new tag ships each release.

## Step 1 — Download the release assets

```bash
TAG="v0.4.0"   # substitute the release tag you're verifying
gh release download "$TAG" --repo zircote/cdc-error-handling --dir ./dl
cd ./dl
ls
# cdc-error-handling-<version>.tar.gz
# cdc-error-handling-<version>-sbom.cdx.json
# cdc-error-handling-<version>-checksums.txt
# marketplace.json.cosign.bundle
```

## Step 2 — Verify the checksum

```bash
sha256sum -c cdc-error-handling-*-checksums.txt
```

This only proves the download wasn't corrupted or truncated — it says
nothing about who built it. That's what the attestation is for.

## Step 3 — Verify the provenance attestation

```bash
gh attestation verify cdc-error-handling-*.tar.gz \
  --repo zircote/cdc-error-handling \
  --signer-workflow zircote/cdc-error-handling/.github/workflows/release.yml \
  --predicate-type https://slsa.dev/provenance/v1
```

A passing verification looks like:

```
Loaded digest sha256:... for file://cdc-error-handling-0.4.0.tar.gz
Loaded 1 attestation from GitHub API
✓ Verification succeeded!
```

This proves the tarball was built by `zircote/cdc-error-handling`'s own
`release.yml` workflow, from the commit the release tag points at, and hasn't
been modified since. `gh attestation verify` exits non-zero on any mismatch
— treat a non-zero exit as a reason not to trust the artifact.

## Step 4 — Verify the SBOM attestation

```bash
gh attestation verify cdc-error-handling-*.tar.gz \
  --repo zircote/cdc-error-handling \
  --signer-workflow zircote/cdc-error-handling/.github/workflows/release.yml \
  --predicate-type https://cyclonedx.org/bom
```

Proves the tarball is bound to the CycloneDX SBOM published alongside it
(`cdc-error-handling-*-sbom.cdx.json`).

## Step 5 — Verify the quality-gate verdicts

Each gate (SAST, SCA, IaC/license, Semgrep, secrets, manifest review) is
attested separately, bound to the same tarball digest:

```bash
for pt in sast sca iac-license semgrep secrets manifest; do
  gh attestation verify cdc-error-handling-*.tar.gz \
    --repo zircote/cdc-error-handling \
    --signer-workflow zircote/cdc-error-handling/.github/workflows/release.yml \
    --predicate-type "https://zircote.github.io/attestations/${pt}/v1"
done
```

A passing verification proves the gate *ran and recorded a verdict* bound to
this exact tarball — read the predicate body (Step 8) for the verdict itself.

## Step 6 — Verify the OpenVEX disposition

```bash
gh attestation verify cdc-error-handling-*.tar.gz \
  --repo zircote/cdc-error-handling \
  --signer-workflow zircote/cdc-error-handling/.github/workflows/release.yml \
  --predicate-type https://openvex.dev/ns/v0.2.0
```

## Step 7 — Verify the catalog signature

The `marketplace.json` catalog is a blob (not an OCI image), so it's signed
with cosign keyless rather than `gh attestation verify`:

```bash
cosign verify-blob .claude-plugin/marketplace.json \
  --bundle marketplace.json.cosign.bundle \
  --certificate-identity-regexp '^https://github\.com/zircote/cdc-error-handling/\.github/workflows/release\.yml@' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

Note this checks the catalog file in your local checkout of the repo
(`.claude-plugin/marketplace.json`), not a release asset — download or
checkout the repo at the release tag first if you don't already have it.

## Step 8 (optional) — Inspect a predicate body

To see the actual details bound into any attestation (build provenance shown
here; substitute any predicate type from the steps above):

```bash
gh attestation verify cdc-error-handling-*.tar.gz \
  --repo zircote/cdc-error-handling \
  --signer-workflow zircote/cdc-error-handling/.github/workflows/release.yml \
  --predicate-type https://slsa.dev/provenance/v1 \
  --format json | jq '.[0].verificationResult.statement.predicate'
```

## What this does and doesn't prove

- **Proves**: this exact file was produced by this repository's release
  workflow from a specific commit, and hasn't been altered since.
- **Doesn't prove**: the code is bug-free, or that the maintainer's account
  or the GitHub Actions runner it built on wasn't itself compromised. See
  [SECURITY.md](../../SECURITY.md) for the full threat-model caveat.

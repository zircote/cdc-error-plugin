---
id: error-handling-security-policy
type: semantic
created: 2026-07-03T10:00:27-04:00
---

# Security Policy

## Reporting a Vulnerability

Please do **not** open a public GitHub issue for security vulnerabilities.

Report security issues using
[GitHub private vulnerability reporting](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing/privately-reporting-a-security-vulnerability)
for this repository (Security → Advisories → "Report a vulnerability"), or by
emailing the maintainer directly.

---

## Where enforcement lives

Claude Code does **not** verify plugin signatures or attestations at install
time yet (tracked upstream:
[anthropics/claude-code#30727](https://github.com/anthropics/claude-code/issues/30727)).
This marketplace cannot rely on the installer to refuse an unverified plugin.
Enforcement lives at these points instead:

1. **CI on every push/PR** (`.github/workflows/ci.yml`) — `claude plugin
   validate . --strict` on both `plugin.json` and `marketplace.json`,
   `actionlint` on the workflow files themselves, and `pin-check` (every
   `uses:` in `.github/workflows` must be a full 40-char commit SHA, never a
   mutable tag or branch).
2. **Fail-closed release verification** (`.github/workflows/release.yml`) —
   the release tarball is attested with SLSA build provenance, a CycloneDX
   SBOM, six quality-gate verdicts (SAST, SCA, IaC/license, Semgrep, secrets,
   manifest review), and an OpenVEX disposition; the `marketplace.json`
   catalog is cosign-signed. Every attestation and the catalog signature are
   re-verified in the same run before the tag-gated publish step. A tag
   publishes nothing that didn't just pass its own verification.
3. **Documented consumer verification** — the commands below let anyone
   re-check a release from a clean workstation before trusting it.

> **Every gate is risk-reducing, not risk-eliminating.** A passing
> verification proves the tarball was built by this repository's own release
> workflow from a specific commit, untampered after signing. It does not
> certify the plugin's contents are free of bugs or the skills it teaches are
> correct for your use case.

---

## Verify a release

Each release tarball carries GitHub's Sigstore-backed (keyless, OIDC)
attestations — no long-lived signing keys, and anyone can re-verify. Unlike a
multi-repo org, this plugin has no separate central signer: this repository's
own `release.yml` workflow produces every attestation and the catalog
signature. Pin `--signer-workflow` to that workflow explicitly rather than
relying on `--repo` alone — `--repo` only proves "signed by some workflow in
this repo," not specifically the release workflow.

Prerequisites: [GitHub CLI](https://cli.github.com/) `gh` ≥ 2.49.0,
authenticated (`gh auth login`); [`cosign`](https://github.com/sigstore/cosign)
for the catalog signature.

```bash
TARBALL="cdc-error-plugin-0.4.0.tar.gz"   # substitute the downloaded release asset
REPO="zircote/cdc-error-plugin"
SIGNER="zircote/cdc-error-plugin/.github/workflows/release.yml"
```

### 1. SLSA build provenance + CycloneDX SBOM

```bash
gh attestation verify "$TARBALL" --repo "$REPO" --signer-workflow "$SIGNER" \
  --predicate-type https://slsa.dev/provenance/v1

gh attestation verify "$TARBALL" --repo "$REPO" --signer-workflow "$SIGNER" \
  --predicate-type https://cyclonedx.org/bom
```

### 2. Quality-gate verdicts

```bash
for pt in sast sca iac-license semgrep secrets manifest; do
  gh attestation verify "$TARBALL" --repo "$REPO" --signer-workflow "$SIGNER" \
    --predicate-type "https://zircote.github.io/attestations/${pt}/v1"
done
```

### 3. OpenVEX disposition

```bash
gh attestation verify "$TARBALL" --repo "$REPO" --signer-workflow "$SIGNER" \
  --predicate-type https://openvex.dev/ns/v0.2.0
```

A passing verification looks like:

```
Loaded digest sha256:... for file://cdc-error-plugin-0.4.0.tar.gz
Loaded 1 attestation from GitHub API
✓ Verification succeeded!
```

A failed verification exits non-zero. **Treat any verification failure as a
supply-chain integrity issue — do not install or use the artifact.**

Also verify the published checksum, included as a release asset alongside the
tarball:

```bash
sha256sum -c cdc-error-plugin-*-checksums.txt   # substitute the downloaded checksums file
```

---

## Verify the catalog signature

The `marketplace.json` catalog is a blob (not an OCI image), signed with
**cosign keyless** (Sigstore Fulcio/Rekor) by this repository's own
`release.yml` workflow. Verify the downloaded catalog against its detached
bundle (also a release asset):

```bash
cosign verify-blob .claude-plugin/marketplace.json \
  --bundle marketplace.json.cosign.bundle \
  --certificate-identity-regexp '^https://github\.com/zircote/cdc-error-plugin/\.github/workflows/release\.yml@' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

See [docs/how-to/verify-release.md](docs/how-to/verify-release.md) for a
narrative walkthrough.

---

## What the attestations prove

| Attestation | Predicate type | What it proves |
| --- | --- | --- |
| SLSA build provenance | `https://slsa.dev/provenance/v1` | The tarball was built by this repo's release workflow from a specific commit, untampered since |
| CycloneDX SBOM | `https://cyclonedx.org/bom` | The tarball is bound to a bill of materials |
| SAST | `.../attestations/sast/v1` | CodeQL ran over this repo's workflows and recorded a verdict |
| SCA | `.../attestations/sca/v1` | OSV-Scanner ran and recorded a verdict (no dependency manifests currently exist, so this asserts an empty, non-vulnerable dependency graph) |
| IaC / license | `.../attestations/iac-license/v1` | Trivy ran (misconfig + license) and recorded a verdict |
| Semgrep | `.../attestations/semgrep/v1` | Semgrep ran and recorded a verdict |
| Secrets | `.../attestations/secrets/v1` | Gitleaks + TruffleHog ran and recorded a verdict |
| Manifest review | `.../attestations/manifest/v1` | `marketplace.json`/`plugin.json` passed structural integrity review |
| OpenVEX | `https://openvex.dev/ns/v0.2.0` | Vulnerability disposition recorded for this release |
| Catalog signature | cosign keyless blob signature | The `marketplace.json` catalog blob is the one this repo published |

> **Signed ≠ passed.** A passing verification proves the gate *ran and
> recorded a verdict* bound to the subject digest. Read the predicate body
> (`gh attestation verify ... --format json`) for the verdict itself.

---

## Supply-chain security posture

- Every GitHub Action referenced in `.github/workflows/` is pinned to a full
  40-character commit SHA, never a mutable tag or branch. The `pin-check` CI
  job enforces this on every push and PR.
- The release pipeline is fail-closed: every attestation and the catalog
  signature must verify before a tag's publish step runs. There is no path
  from build to publish that bypasses verification.
- This is a single solo-maintained plugin repository, not a multi-repo org.
  There is no fleet of GitHub Apps minting release tokens (the default
  `GITHUB_TOKEN`, scoped per job, is used throughout) and no separate central
  signing seam — this repo's own `release.yml` is the signer for every
  attestation and the catalog signature. Verification still pins
  `--signer-workflow` to that workflow explicitly: `--repo` alone only
  proves "signed by some workflow in this repo," not specifically the
  release workflow.
- ShellCheck is deliberately not wired as a gate: this repo has no `.sh`
  scripts to check (its only executable logic lives in `.github/workflows/*.yml`
  `run:` blocks). Wiring a gate with nothing to scan would produce a
  permanently-empty attestation, which asserts nothing and would be
  misleading.

For the reasoning behind this pipeline's scope and design (why these gates,
why no central signer, what it does and doesn't close), see
[docs/explanation/attested-releases.md](docs/explanation/attested-releases.md).

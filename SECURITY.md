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
   the release tarball is attested with GitHub's SLSA build-provenance
   attestation and re-verified in the same run before the tag-gated publish
   step. A tag publishes nothing that didn't just pass its own verification.
3. **Documented consumer verification** — the command below lets anyone
   re-check a release from a clean workstation before trusting it.

> **Every gate is risk-reducing, not risk-eliminating.** A passing
> verification proves the tarball was built by this repository's own release
> workflow from a specific commit, untampered after signing. It does not
> certify the plugin's contents are free of bugs or the skills it teaches are
> correct for your use case.

---

## Verify a release

Each release tarball carries a [SLSA build-provenance](https://slsa.dev/provenance/v1)
attestation, produced by GitHub's Sigstore-backed (keyless, OIDC) attestation
infrastructure — no long-lived signing keys, and anyone can re-verify.

Prerequisites: [GitHub CLI](https://cli.github.com/) `gh` ≥ 2.49.0,
authenticated (`gh auth login`).

```bash
TARBALL="cdc-error-handling-0.4.0.tar.gz"   # substitute the downloaded release asset
REPO="zircote/cdc-error-handling"

gh attestation verify "$TARBALL" --repo "$REPO" \
  --predicate-type https://slsa.dev/provenance/v1
```

A passing verification looks like:

```
Loaded digest sha256:... for file://cdc-error-handling-0.4.0.tar.gz
Loaded 1 attestation from GitHub API
✓ Verification succeeded!
```

A failed verification exits non-zero. **Treat any verification failure as a
supply-chain integrity issue — do not install or use the artifact.**

Also verify the published checksum, included as a release asset alongside the
tarball:

```bash
sha256sum -c cdc-error-handling-0.4.0-checksums.txt
```

See [docs/how-to/verify-release.md](docs/how-to/verify-release.md) for a
narrative walkthrough.

---

## Supply-chain security posture

- Every GitHub Action referenced in `.github/workflows/` is pinned to a full
  40-character commit SHA, never a mutable tag or branch. The `pin-check` CI
  job enforces this on every push and PR.
- The release pipeline is fail-closed: the SLSA provenance attestation must
  verify before a tag's publish step runs. There is no path from build to
  publish that bypasses verification.
- This is a single solo-maintained plugin repository, not a multi-repo
  org — there is no fleet of GitHub Apps, no shared signing seam, and no SBOM
  or VEX pipeline. Those exist to attest a dependency tree and coordinate
  many repos; this plugin has neither.

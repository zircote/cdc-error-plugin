---
id: explanation-why-attested-releases
type: semantic
created: 2026-07-03T00:00:00-04:00
diataxis_type: explanation
title: Why this plugin ships an attested release pipeline
---

# Why this plugin ships an attested release pipeline

Every tagged release of this plugin runs through eighteen CI jobs before a
GitHub Release exists: a reproducible build, SLSA provenance, a CycloneDX
SBOM, six quality-gate verdicts, an OpenVEX disposition, and a cosign-signed
marketplace catalog. That is a lot of machinery for a markdown/JSON skill
bundle with no runtime dependencies. This explains why it's there, and why
it looks the way it does.

## The gap it doesn't close

Claude Code does not verify plugin signatures or attestations at install
time yet ([anthropics/claude-code#30727](https://github.com/anthropics/claude-code/issues/30727)).
Nothing in this pipeline stops a user from installing an unverified plugin
— the installer doesn't ask. So why build fail-closed verification for a
gate the installer doesn't enforce?

Because enforcement doesn't have to live in the installer to be real:

1. **It's opt-in, not automatic, but it's genuine.** Anyone can independently
   re-verify a release from a clean workstation (see
   [docs/how-to/verify-release.md](../how-to/verify-release.md)) and get a
   real answer, not a marketing claim. The absence of install-time
   enforcement is a documented upstream gap, not a reason to skip building
   the thing that closes it the moment Claude Code catches up.
2. **The pipeline is a real gate on this repo, today.** Every attestation
   must verify before the `publish` job runs — there is no code path from a
   broken build to a public release. This isn't hypothetical: the
   `workflow_dispatch` dry-run for `v0.4.0` caught a real bug (`gate-sast`
   double-uploading to code scanning) before any tag was cut. The gate
   protects the maintainer from shipping a broken release, independent of
   whether any consumer ever runs `gh attestation verify`.
3. **It's the same shape a supply-chain-conscious org already trusts.**
   Building this pipeline against SLSA provenance, CycloneDX, and OpenVEX —
   rather than a bespoke scheme — means a consumer who already verifies
   attestations from other software can point the same tools at this
   plugin and get a real answer, without learning a new format.

## Why six gates and an SBOM for a repo with no dependencies

This plugin has no package.json, no Cargo.toml, no dependency tree to
speak of. SCA (OSV-Scanner) and the SBOM against an empty dependency graph
might look like theater. They aren't, for two reasons:

- **"No dependencies" is a claim, and claims can rot.** An SBOM and an SCA
  gate don't just report on today's dependency graph — they're wired to
  run on every future release. The day this plugin adds a dependency (a
  build script, an eval harness with its own `package.json`), the gate is
  already there to catch a known-vulnerable version, rather than needing
  to be retrofitted under time pressure.
- **The other five gates (SAST, Trivy, Semgrep, secrets, manifest review)
  scan the one thing this repo genuinely has: its own CI/CD surface.** The
  SAST gate runs CodeQL's `actions` language against the workflow YAML
  itself — the actual attack surface here isn't application code, it's the
  release pipeline's own logic. A bug in `release.yml` that weakens the
  fail-closed verification is exactly the class of defect these gates
  exist to catch.

ShellCheck is the one gate deliberately *not* wired: this repo has no `.sh`
scripts, only YAML `run:` blocks. A gate with nothing to scan would produce
a permanently-empty attestation that asserts nothing real — worse than no
gate, because it looks like coverage without being coverage. See
[SECURITY.md](../../SECURITY.md) for the full list of what each gate
actually checks.

## Why there's no central signer

The reference architecture this pipeline was adapted from
([modeled-information-format/claude-code-plugins](https://github.com/modeled-information-format/claude-code-plugins))
signs from a *separate* repository — a shared reusable workflow that many
caller repos trust, so that under SLSA Build L3 the certificate identity is
the central signer, not any individual artifact repo. That separation
exists to let one org's many repos share a single audited signer instead of
each re-implementing signing logic.

This repository has no fleet of sibling repos to share a signer with. It's
a single, solo-maintained plugin. Building a separate signer repo for an
audience of one would add a second thing to keep secure and no additional
protection — the trust boundary is already "whoever can push to this repo,"
identical with or without a separate signer. So every attestation and the
catalog signature are produced by this same `release.yml`, and verification
pins `--signer-workflow` to that one workflow explicitly rather than a
distinct central identity. If this plugin ever becomes one of several repos
under a shared org that wants a common signer, that's a real architectural
change worth making then — not a gap to paper over now.

## What this doesn't prove

An attestation proves a gate *ran and recorded a verdict* bound to a
specific artifact digest. It does not prove the code is bug-free, that the
maintainer's account wasn't compromised, or that the GitHub Actions runner
itself was trustworthy at build time. Every gate here is risk-reducing, not
risk-eliminating — see [SECURITY.md](../../SECURITY.md) for the complete
threat-model caveat.

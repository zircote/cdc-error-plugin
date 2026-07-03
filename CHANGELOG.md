---
id: error-handling-changelog
type: episodic
created: 2026-07-03T10:00:27-04:00
---

# Changelog

All notable changes to this plugin are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this plugin adheres to [Semantic Versioning](https://semver.org/).

Plugin version (`.claude-plugin/plugin.json`) and marketplace catalog version
(`.claude-plugin/marketplace.json`) are versioned independently. This log
tracks the plugin.

## [Unreleased]

### Added

- `docs/explanation/attested-releases.md`: why the release pipeline has the
  scope and shape it does (six gates + SBOM + VEX + cosign for a repo with
  no dependency tree, no central signer).
- MIF Level 1 frontmatter on the 6 docs that lacked it
  (`getting-started.md`, `add-cdc-err-to-cli.md`, `run-evals.md`,
  `dual-consumer.md`, `skill-cooperation.md`, `reference/index.md`). All 11
  docs in the repo now pass `mif-validate --level 1`.

### Fixed

- `getting-started.md`: stale `/plugin install zircote/error-handling`
  install command; stale claim that the skill picker shows only two
  skills.
- `dual-consumer.md`, `README.md`: stale "two sibling skills" link text
  and "Both skills decline..." heading, both predating the `cdc-handle`
  skill added in 0.3.0.
- `reference/index.md`: plugin manifest table still showed version
  `0.2.0`; added the marketplace.json manifest, previously undocumented.
- `run-evals.md`: `cdc-handle` was missing from the eval-inspection
  commands, `/autoresearch` invocations, and quality-ratio table.
- `skills/cdc-err/SKILL.md`, `skills/cdc-err/references/envelope.md`,
  `README.md`: the source post was silent, not exclusion-free, on
  streaming/multi-error output ("RFC 9457 does not cover streaming or
  multi-error aggregation as elegantly as SARIF does") — added that
  caveat and the post's own `errors[]`/JSON-Lines suggestion for that
  case, verified by re-fetching the source post directly.
- `skills/cdc-review/SKILL.md`, `README.md`: removed a fabricated
  citation, `C-PANIC-FREE` — no such item exists in the Rust API
  guidelines (verified against the guidelines' own checklist page); the
  underlying advice against panicking in library code is real Rust
  community convention, just not a numbered guideline. Also fixed
  `SKILL.md`'s Effective Java citation from "item 73-77" to "items
  69-77" (verified against the book's actual Exceptions chapter table of
  contents), matching what `README.md`/`java.md` already said correctly.

## [0.4.0] - 2026-07-03

### Added

- `.claude-plugin/marketplace.json`: this repository now doubles as a
  single-plugin marketplace, referencing the plugin via a local `./` source.
- `homepage` and `repository` fields on the plugin manifest.
- CI (`.github/workflows/ci.yml`): pin-check
  (`zgosalvez/github-actions-ensure-sha-pinned-actions`, every action `uses:`
  pinned to a full 40-char commit SHA), actionlint
  (`reviewdog/action-actionlint`), and `claude plugin validate . --strict`.
- Release pipeline (`.github/workflows/release.yml`): reproducible
  `git archive` tarball, SLSA build provenance via
  `actions/attest-build-provenance`, a CycloneDX SBOM (`anchore/sbom-action`
  + `actions/attest-sbom`), six quality-gate verdicts each seam-attested as a
  signed custom predicate bound to the tarball digest (SAST via CodeQL, SCA
  via OSV-Scanner, IaC/license via Trivy, Semgrep, secrets via Gitleaks +
  TruffleHog, manifest structural review), an OpenVEX disposition
  attestation (`.vex/openvex.json`), and a cosign keyless signature over the
  `marketplace.json` catalog. All attestations and the catalog signature are
  fail-closed re-verified in the same run before the tag-gated publish step.
  This repo has no separate central signer, so verification uniformly pins
  `--repo` rather than a distinct `--signer-workflow`. ShellCheck is
  deliberately not wired: this repo has no `.sh` scripts to scan.
- `SECURITY.md` documenting the verification model, the full attestation
  table, and how to independently verify every attestation and the catalog
  signature.
- `docs/how-to/verify-release.md`.
- `.vex/openvex.json`: OpenVEX vulnerability-disposition document.

## [0.3.0] - 2026-05-21

### Added

- `cdc-handle` skill: LLM-agent consumer-side interpretation of `cdc-err`
  RFC 9457 envelopes in `tool_result` payloads (parse / decide / act).
- Diátaxis documentation set (`docs/tutorials`, `docs/how-to`,
  `docs/explanation`, `docs/reference`).

### Changed

- Plugin metadata and README updated to describe three sibling skills
  (`cdc-err`, `cdc-review`, `cdc-handle`) instead of two.

## [0.2.0] - 2026-05-20

### Added

- Initial plugin scaffold with the `cdc-err` (RFC 9457 dual-consumer CLI
  error envelopes) and `cdc-review` (source-code error-handling review)
  skills.

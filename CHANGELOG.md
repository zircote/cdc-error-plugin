# Changelog

All notable changes to this plugin are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this plugin adheres to [Semantic Versioning](https://semver.org/).

Plugin version (`.claude-plugin/plugin.json`) and marketplace catalog version
(`.claude-plugin/marketplace.json`) are versioned independently. This log
tracks the plugin.

## [Unreleased]

## [0.4.0] - 2026-07-03

### Added

- `.claude-plugin/marketplace.json`: this repository now doubles as a
  single-plugin marketplace, referencing the plugin via a local `./` source.
- `homepage` and `repository` fields on the plugin manifest.
- CI (`.github/workflows/ci.yml`): pin-check (every action `uses:` pinned to a
  full 40-char commit SHA), actionlint, and `claude plugin validate .`.
- Release pipeline (`.github/workflows/release.yml`): reproducible
  `git archive` tarball, SLSA build provenance via
  `actions/attest-build-provenance`, fail-closed `gh attestation verify`
  before publish, tag-gated GitHub Release with checksums.
- `SECURITY.md` documenting the verification model and how to independently
  verify a release's provenance attestation.
- `docs/how-to/verify-release.md`.

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

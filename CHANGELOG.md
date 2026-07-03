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

- `docs/adr/`: Architectural Decision Records (Structured MADR format) for
  every architecturally significant decision made this session — single-repo
  plugin-and-marketplace architecture, full-tier release attestation scope,
  self-signed attestations (no central signer), default `GITHUB_TOKEN` for
  release publishing, self-referential catalog SHA-pinning via a
  PAT-authenticated post-release job, and the GitHub Flow branch model.
- `.github/workflows/release.yml`: a `pin-catalog` job, gated to real tag
  pushes, that re-pins `marketplace.json`'s `error-handling` entry to the
  just-released commit's SHA and bumps the catalog's patch version,
  authenticated via a dedicated fine-grained PAT (`CATALOG_PIN_TOKEN`) since
  the default token can't bypass `main`'s branch protection the way an
  authenticated admin push can.
- `docs/how-to/catalog-pinning.md`: runbook for the automated catalog-pin
  flow and its manual fallback.

## [0.4.1] - 2026-07-03

### Changed

- Repository renamed `zircote/cdc-error-handling` -> `zircote/cdc-error-plugin`.
  Updated every reference to the old slug: `plugin.json`'s `homepage`/
  `repository`, `release.yml`'s cosign signer-identity regex (would have
  silently broken the next release's self-verification if missed, since
  it's hardcoded to the repo path), and every install/verify command
  example across the docs.
- Marketplace catalog renamed `error-handling` -> `cdc-errors` in
  `marketplace.json`'s top-level `name` (the plugin's own `name` is
  unchanged, still `error-handling` — these are independent identifiers).
  Updated every `/plugin install error-handling@<marketplace>` example and
  the `extraKnownMarketplaces` config key to match.
- Default branch renamed `develop` -> `main`; adopted GitHub Flow (single
  trunk, feature branches merge back via PR). `ci.yml`'s triggers updated
  to match. Branch protection added on `main`: `pin-check`, `actionlint`,
  and `claude plugin validate` required (strict/up-to-date mode), force-push
  and branch deletion disabled. No required PR review (solo-maintained,
  no other collaborators to review) and `enforce_admins: false` (admin can
  still push directly).

### Added

- `docs/README.md`: an index of all Diátaxis-organized docs, one section
  per quadrant (tutorial/how-to/reference/explanation) plus links to the
  root-level documents.
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

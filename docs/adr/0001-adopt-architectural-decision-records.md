---
id: adr-0001-adopt-architectural-decision-records
type: adr
created: 2026-07-03T00:00:00-04:00
title: "Adopt Architectural Decision Records for This Project"
description: "Establish docs/adr/ as the durable record of architecturally significant, hard-to-reverse decisions this repo makes, in Structured MADR format, after several such decisions (attestation scope, signer topology, release auth, catalog pinning, branch model) were made in a single session with no record beyond commit messages and CHANGELOG entries."
category: process
tags:
  - adr
  - process
  - documentation
status: accepted
updated: 2026-07-03
author: zircote
project: error-handling-plugin
audience:
  - developers
  - architects
related: []
---

# ADR-0001: Adopt Architectural Decision Records for This Project

## Status

Accepted

## Context

### Background and Problem Statement

Over one working session, this repo went from a two-skill plugin scaffold to
a single-repo plugin-plus-marketplace with a full attested release pipeline
(SLSA provenance, SBOM, six quality gates, OpenVEX, cosign catalog
signature), a GitHub Flow branch model, and an automated post-release
catalog-pinning step authenticated by a dedicated PAT. Several of these
decisions had genuine alternatives that were weighed and rejected — full vs.
minimal attestation tiers, a central signer vs. a self-signed one, GitHub
Apps vs. the default `GITHUB_TOKEN`, a PR-based vs. direct-push vs.
PAT-based catalog write-back. None of that reasoning has a durable home:
`CHANGELOG.md` records *what* shipped, not *why one option beat the others*,
and commit messages are read once, in passing, by whoever is looking at that
specific diff.

This repo is solo-maintained. The person who most needs the "why" behind a
choice like "why does this repo have no central attestation signer" is the
same person who made the call, revisiting it months later with no memory of
the tradeoff. A future Claude Code session picking up this repo cold has the
same problem, at a shorter timescale.

### Current Limitations

1. **No record of rejected alternatives**: `CHANGELOG.md`'s "Changed" entries
   describe the shipped state, not the options that lost and why.
2. **No queryable decision status**: nothing marks a past decision as still
   in force, superseded, or worth revisiting when circumstances change (e.g.
   if this repo ever gains a second local plugin, or joins a multi-repo org).
3. **No place for the risk assessment behind a choice**: e.g. the PAT
   rotation burden accepted in exchange for automated catalog pinning is
   currently only visible by reading `docs/how-to/catalog-pinning.md`'s
   prose, not flagged as a deliberately accepted risk.

## Decision Drivers

### Primary Decision Drivers

1. **Durability**: a decision's rationale shall survive independently of the
   commit or PR that implemented it.
2. **Machine-readability**: an agent picking up this repo shall be able to
   determine, from frontmatter alone, whether a given decision is still in
   force (`status`) and what it relates to (`related`).
3. **Low ceremony for a solo repo**: the format shall not require a review
   board, approval workflow, or multi-person sign-off this repo doesn't have.

### Secondary Decision Drivers

1. **Consistency with existing documentation conventions**: this repo
   already uses MIF L1 frontmatter (`id`/`type`/`created`) across every doc
   under `docs/`; the ADR format should compose with that rather than
   introduce an unrelated scheme.

## Considered Options

### Option 1: Rely on CHANGELOG.md only

**Description**: Keep recording decisions as CHANGELOG entries under
"Changed", with prose explaining rationale inline.

**Advantages**:

- Already the established convention; zero new process.

**Disadvantages**:

- CHANGELOG entries are terse by convention (Keep a Changelog is a
  human-readable release log, not a decision record) and don't have a slot
  for "options considered and rejected."
- No `status` field — a CHANGELOG entry can't mark itself superseded.

**Risk Assessment**:

- **Technical Risk**: Low.
- **Schedule Risk**: None.
- **Ecosystem Risk**: None.

**Disqualifying Factor**: no structured way to record rejected alternatives
or mark a decision's current status.

### Option 2: Rely on git commit messages only

**Description**: Trust that a sufficiently detailed commit message (this
session's commits already run long, e.g. the catalog-pin and attestation
commits) is enough of a record.

**Advantages**:

- No new files or process; already happening.

**Disadvantages**:

- Commit messages are read once, attached to a diff a reader is already
  looking at — they aren't discoverable by browsing the repo, and `git log`
  isn't a place someone looks to answer "why don't we have a central
  signer?" six months from now.
- No cross-linking between related decisions.

**Risk Assessment**:

- **Technical Risk**: Low.
- **Schedule Risk**: None.
- **Ecosystem Risk**: None.

**Disqualifying Factor**: not discoverable independent of the commit that
made the change.

### Option 3: Adopt Architectural Decision Records (Structured MADR)

**Description**: A `docs/adr/` directory of one file per decision, Structured
MADR format (context, decision drivers, considered options with risk
assessment, decision, consequences, audit trail), with MIF-conformant
frontmatter matching the convention already used across `docs/`.

**Advantages**:

- Purpose-built for exactly this problem: one architecturally significant
  decision per record, with the alternatives and the accepted cost made
  explicit.
- `status` lifecycle (`proposed`/`accepted`/`deprecated`/`superseded`) gives
  a decision a queryable state instead of leaving it implicit.
- Composes with this repo's existing MIF-frontmatter convention rather than
  introducing a competing scheme.

**Disadvantages**:

- More ceremony per decision than a CHANGELOG line — not every change
  warrants an ADR (only architecturally significant, hard-to-reverse ones
  with genuine alternatives do).

**Risk Assessment**:

- **Technical Risk**: Low. Plain markdown, no tooling dependency to author.
- **Schedule Risk**: Low. Retrofitting the decisions already made this
  session is a bounded, one-time cost.
- **Ecosystem Risk**: None.

## Decision

We adopt Architectural Decision Records at `docs/adr/`, in Structured MADR
format, for every architecturally significant, hard-to-reverse decision this
repo makes going forward — and retrofit the decisions already made in this
session (ADR-0002 through ADR-0007). Not every change gets an ADR: routine
fixes, doc corrections, and version bumps stay in `CHANGELOG.md`. An ADR is
warranted specifically when there were genuine alternatives and the choice
is expensive to reverse.

## Consequences

### Positive

1. **Rationale survives the commit that implemented it** — readable
   independent of `git log` or a specific diff.
2. **Future sessions (human or agent) can query decision status** via
   frontmatter rather than re-deriving it from prose.

### Negative

1. **Retrofitting existing decisions is a one-time cost** paid in this same
   session, and every future architecturally significant decision now
   carries the overhead of a full ADR rather than a CHANGELOG line.

### Neutral

1. **Numbering is sequential and permanent** — an ADR is never renumbered or
   deleted, only superseded by a newer one that links back to it.

## Decision Outcome

Measured by: every architecturally significant decision made in this session
has a corresponding ADR (ADR-0002 through ADR-0007) before this ADR set is
considered complete; going forward, a new architecturally significant
decision without an ADR is treated as a gap to fill, not an acceptable
omission.

## Related Decisions

- [ADR-0002: Single-Repo Plugin-and-Marketplace Architecture](0002-single-repo-plugin-and-marketplace.md)
- [ADR-0003: Full-Tier Attestation Scope for the Release Pipeline](0003-full-tier-attestation-scope.md)
- [ADR-0004: Self-Signed Attestations, No Central Signer](0004-no-central-attestation-signer.md)
- [ADR-0005: Default GITHUB_TOKEN for Release Publishing](0005-default-token-for-release-auth.md)
- [ADR-0006: Self-Referential Catalog SHA-Pinning via PAT-Authenticated Automation](0006-self-referential-catalog-pinning.md)
- [ADR-0007: GitHub Flow Branch Model](0007-github-flow-branch-model.md)

## Links

- [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) - the complementary, human-readable release log this repo also keeps.
- [Documenting Architecture Decisions (Michael Nygard)](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions) - the original ADR proposal this format extends.

## More Information

- **Date:** 2026-07-03
- **Source:** retrospective, prompted by a request to establish ADRs after several undocumented architectural decisions accumulated in one session.
- **Related ADRs:** 0002, 0003, 0004, 0005, 0006, 0007

## Audit

### 2026-07-03

**Status:** Compliant

**Findings:**

| Finding | Files | Lines | Assessment |
| --- | --- | --- | --- |
| ADR practice established with a retrofit set covering all decisions made this session | `docs/adr/0001`-`0007` | - | accepted |

**Summary:** ADR practice adopted; the retrofit set (ADR-0002 through
ADR-0007) covers every architecturally significant decision identified in
this session's review.

**Action Required:** None. Apply the practice going forward.

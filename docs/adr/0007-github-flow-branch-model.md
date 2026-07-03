---
id: adr-0007-github-flow-branch-model
type: adr
created: 2026-07-03T00:00:00-04:00
title: "GitHub Flow Branch Model"
description: "The repository's default branch was renamed develop -> main and a single-trunk GitHub Flow model was adopted (feature branches merge back via PR, no separate develop/release branches), with branch protection requiring pin-check/actionlint/claude-plugin-validate before merge and no required PR review."
category: repository-architecture
tags:
  - adr
  - architecture
  - git-workflow
  - branch-protection
status: accepted
updated: 2026-07-03
author: zircote
project: error-handling-plugin
technologies:
  - github
audience:
  - developers
audience_note: solo-maintained repo, no other collaborators
related:
  - 0006-self-referential-catalog-pinning.md
---

# ADR-0007: GitHub Flow Branch Model

## Status

Accepted

## Context

### Background and Problem Statement

The repository's original default branch was named `develop`, with no
`main`/`master` branch at all — a naming holdover with no corresponding
git-flow structure (no `release`/`hotfix` branches ever existed here). The
explicit request was to rename `develop` to `main` and "make it github
flow" — establishing a deliberate single-trunk model rather than leaving the
branch name as an artifact of however the repo happened to be initialized.

Separately, this repo is solo-maintained with no other collaborators. A
branch-protection model built for a team (mandatory peer review before
merge) would be actively counterproductive here — it would lock the
repository, since there is no second person to grant the required approval.

### Current Limitations

1. **`develop` as the default branch with no `main`** is a git-flow-shaped
   name without git-flow's actual structure — misleading to anyone (human
   or agent) who assumes `develop` implies a separate `main`/`release`
   branch exists.
2. **No branch protection existed at all** prior to this decision — nothing
   enforced that CI had passed before a change landed on the trunk.
3. **`.github/workflows/ci.yml`'s triggers, `release.yml`'s tag-push
   semantics, and every doc's install instructions** were all still written
   against `develop`, needing to move in lockstep with the rename.

## Decision Drivers

### Primary Decision Drivers

1. **The default branch name shall reflect the actual branching model in
   use** (single trunk), not an inherited name implying a structure that
   was never built.
2. **Branch protection shall require CI to pass before merge**, closing the
   gap where nothing previously enforced that.
3. **Branch protection shall not require peer review** — this repo has no
   second collaborator to provide it, and requiring it would deadlock every
   merge.

### Secondary Decision Drivers

1. **The repo owner shall retain the ability to push directly for quick,
   solo fixes** — the established pattern throughout this session was
   direct pushes for small, already-verified changes, not a PR for every
   single commit.

## Considered Options

### Option 1: Keep `develop`, add `main` as a separate integration branch (git-flow-adjacent)

**Description**: Introduce an actual `main` branch distinct from `develop`,
with `develop` as the working trunk and periodic merges to `main` for
releases — closer to the structure the original `develop` name implied.

**Advantages**:

- Matches the branch name's original implication.

**Disadvantages**:

- Adds a merge/promotion step (`develop` -> `main`) with no corresponding
  need: this repo has no release-branch, hotfix-branch, or parallel-version
  requirement that git-flow's extra structure exists to serve.
- Directly contradicts the explicit request to "make it github flow" (a
  single-trunk model by definition).

**Risk Assessment**:

- **Technical Risk**: Low.
- **Schedule Risk**: Low.
- **Ecosystem Risk**: Low, but adds process with no corresponding benefit.

**Disqualifying Factor**: contradicts the explicit request and adds
unneeded structure.

### Option 2: Single trunk (`main`), GitHub Flow, required status checks, no required review

**Description**: Rename `develop` to `main` (GitHub's native branch-rename,
preserving the default-branch pointer and any existing PRs), update every
branch reference in the repo (`ci.yml` triggers, docs), and add branch
protection requiring `pin-check`/`actionlint`/`claude plugin validate`
before merge, with `required_pull_request_reviews: null` and
`enforce_admins: false`.

**Advantages**:

- Matches both the explicit request and the repo's actual solo-maintainer
  reality: CI must pass, but the owner isn't locked out by a review
  requirement no one else can satisfy.
- `enforce_admins: false` preserves the ability to push directly for
  quick, already-verified fixes — the pattern already established this
  session — while still gating PR-based merges on CI.

**Disadvantages**:

- `enforce_admins: false` means required status checks don't actually block
  the repo owner's own direct pushes — a real safety gap if a mistake ships
  without CI having run against that exact commit, mitigated only by the
  discipline of running `actionlint`/`claude plugin validate` locally
  before pushing (which this session did consistently, but the protection
  rule itself doesn't enforce it for admin pushes).

**Risk Assessment**:

- **Technical Risk**: Low. The rename via GitHub's native API is largely
  safe, though see the Audit section below for a bug caught during this
  exact operation.
- **Schedule Risk**: Low.
- **Ecosystem Risk**: Low.

## Decision

We adopt Option 2: single-trunk `main`, GitHub Flow, branch protection
requiring `pin-check`, `actionlint`, and `claude plugin validate` (strict/
up-to-date mode) before a PR merges, force-push and branch deletion
disabled, no required PR review, `enforce_admins: false`.

## Consequences

### Positive

1. **The default branch name now reflects the actual model in use** — no
   implied structure that doesn't exist.
2. **PR-based merges are gated on CI passing**, closing a real gap.
3. **The repo owner retains direct-push ability** for quick, solo,
   already-verified changes — matching how this repo is actually operated.

### Negative

1. **`enforce_admins: false` means the CI gate doesn't protect against the
   admin's own mistakes on a direct push** — only PR-based merges are truly
   gated. This is an accepted tradeoff for a solo maintainer, not an
   oversight.
2. **ADR-0006's catalog-pinning automation needed a dedicated PAT** as a
   direct consequence of this branch protection existing at all — the
   default `GITHUB_TOKEN` isn't treated as the same kind of bypass-eligible
   actor a human admin is.

### Neutral

1. **The rename itself surfaced a real GitHub API bug**: renaming the
   current default branch via `POST .../branches/{branch}/rename` left the
   default-branch pointer on a stale feature branch instead of the newly
   renamed `main`, requiring an explicit follow-up `PATCH
   default_branch=main`. Caught by independently re-checking the API
   response rather than trusting the rename call's own success response —
   worth remembering for any future branch rename on this or another repo.

## Decision Outcome

Measured by: `gh api repos/zircote/cdc-error-plugin --jq '.default_branch'`
confirmed `main` after the corrective `PATCH`; `ci.yml` triggers verified
firing correctly on `main` pushes; a subsequent direct push confirmed
`enforce_admins: false` behaves as intended ("Bypassed rule violations"
notice, push succeeds).

## Related Decisions

- [ADR-0006: Self-Referential Catalog SHA-Pinning via PAT-Authenticated Automation](0006-self-referential-catalog-pinning.md) - required a dedicated PAT specifically because of the branch protection this ADR establishes.

## Links

- [GitHub Flow](https://docs.github.com/en/get-started/using-github/github-flow) - the single-trunk model adopted here.
- [About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches) - `enforce_admins` and required-status-checks semantics this decision relies on.

## More Information

- **Date:** 2026-07-03
- **Source:** explicit user request ("rename develop -> main and make it github-flow").
- **Related ADRs:** 0006

## Audit

### 2026-07-03

**Status:** Compliant

**Findings:**

| Finding | Files | Lines | Assessment |
| --- | --- | --- | --- |
| Rename API left default_branch pointer on wrong branch; caught and corrected | GitHub repo settings | - | fixed |
| ci.yml triggers and doc branch references updated to main | `.github/workflows/ci.yml`, docs | - | accepted |

**Summary:** Branch rename and GitHub Flow adoption completed; one real API
inconsistency caught by independent verification and corrected before it
could cause a broken default-branch state.

**Action Required:** None. Revisit `enforce_admins: false` if this repo ever
gains a second collaborator whose direct pushes should also be CI-gated.

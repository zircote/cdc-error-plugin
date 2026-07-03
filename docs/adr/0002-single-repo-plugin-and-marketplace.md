---
id: adr-0002-single-repo-plugin-and-marketplace
type: adr
created: 2026-07-03T00:00:00-04:00
title: "Single-Repo Plugin-and-Marketplace Architecture"
description: "This repository serves as both the error-handling plugin and its own single-plugin marketplace catalog, rather than splitting the marketplace catalog into a separate repository the way multi-plugin marketplace orgs typically do."
category: repository-architecture
tags:
  - adr
  - architecture
  - marketplace
  - plugin
status: accepted
updated: 2026-07-03
author: zircote
project: error-handling-plugin
technologies:
  - claude-code-plugins
audience:
  - developers
  - architects
related:
  - 0006-self-referential-catalog-pinning.md
---

# ADR-0002: Single-Repo Plugin-and-Marketplace Architecture

## Status

Accepted

## Context

### Background and Problem Statement

This repository began as a plain Claude Code plugin: `.claude-plugin/plugin.json`
at the root, skills auto-discovered under `skills/`, no marketplace manifest
at all. To be installable via `/plugin marketplace add` + `/plugin install`,
a Claude Code plugin needs a marketplace to belong to. The reference
architecture studied for this work
([modeled-information-format/claude-code-plugins](https://github.com/modeled-information-format/claude-code-plugins))
uses a dedicated marketplace repository (`claude-code-plugins`) separate from
each plugin's own repository (e.g. `mif-docs-plugin`), with the marketplace
listing plugins via external `github`/`git-subdir` sources pinned to a ref
and SHA. That pattern exists to let one marketplace catalog many plugins,
potentially maintained by different people, across separate repos.

This repo holds exactly one plugin, maintained by one person, and the
explicit ask was: "This marketplace will serve for now only the one plugin
it holds and will be versioned at both the plugin and the marketplace
level." A separate marketplace repo would mean two repos, two CI pipelines,
and a cross-repo version-sync problem for a marketplace that will only ever
list one thing.

### Current Limitations

1. **No marketplace existed at all** prior to this decision — the plugin was
   only installable by an operator manually pointing Claude Code at a local
   checkout, not via `/plugin marketplace add`.
2. **A multi-repo split has real coordination cost**: keeping a marketplace
   repo's pinned `ref`/`sha` in sync with the plugin repo's own releases is
   exactly the cross-repo problem ADR-0006's catalog-pinning automation
   exists to solve *within* one repo — splitting into two repos would need
   that same automation to also cross a repo boundary.

## Decision Drivers

### Primary Decision Drivers

1. **Single plugin, single maintainer**: the marketplace shall not carry
   coordination overhead sized for a multi-plugin, multi-maintainer org when
   it lists exactly one plugin maintained by one person.
2. **Independent versioning still required**: the plugin's own version and
   the marketplace catalog's version shall remain independently
   incrementable, even though they live in one repo.

### Secondary Decision Drivers

1. **Minimize repo count for a solo maintainer**: fewer repos to keep CI,
   branch protection, and secrets configured consistently across.

## Considered Options

### Option 1: Separate marketplace repo (mirrors the reference org)

**Description**: Create a second repository (e.g. `zircote/claude-plugins`)
whose sole content is `.claude-plugin/marketplace.json`, listing this
plugin via an external `github` source pinned to `ref`+`sha`, exactly as the
reference org's `claude-code-plugins` repo lists `mif-docs-plugin`.

**Advantages**:

- Matches a proven, multi-repo-scale pattern exactly.
- A marketplace repo could later list additional plugins from other repos
  without restructuring.

**Disadvantages**:

- Two repos to maintain (CI, branch protection, secrets) for content that
  will only ever describe one plugin.
- The plugin-release-to-marketplace-repin step (ADR-0006) would need to
  cross a repo boundary — a second repo's `CATALOG_PIN_TOKEN`-equivalent,
  a cross-repo PR or push, more moving parts for the same effective outcome.

**Risk Assessment**:

- **Technical Risk**: Low. The pattern is proven at the reference org.
- **Schedule Risk**: Medium. Doubles the repos needing CI/protection setup.
- **Ecosystem Risk**: Low.

**Disqualifying Factor**: coordination cost disproportionate to a
single-plugin, single-maintainer catalog.

### Option 2: Single-repo dual role (plugin + marketplace)

**Description**: Add `.claude-plugin/marketplace.json` to this same repo,
listing the plugin via a local source (`"./"`, later converted to a
self-referential SHA-pinned `github` source per ADR-0006), with its own
independent `version` field.

**Advantages**:

- One repo, one CI pipeline, one branch-protection configuration.
- Matches the explicit requirement: plugin and marketplace versioned
  independently, without needing a second repo to host that independence.
- The catalog-pinning automation (ADR-0006) stays within one repo's
  permission boundary — no cross-repo token needed.

**Disadvantages**:

- Doesn't generalize automatically if this maintainer ever wants to list a
  second, genuinely external plugin from a different repo in the same
  catalog (though the schema supports mixing local and external entries in
  one `marketplace.json`, so this isn't actually blocked, just unexercised).

**Risk Assessment**:

- **Technical Risk**: Low. The official plugin-marketplace schema
  explicitly supports this shape (`source: "./"` at the marketplace root,
  confirmed against the current official documentation before implementing).
- **Schedule Risk**: Low.
- **Ecosystem Risk**: Low.

## Decision

We adopt the single-repo dual role: this repository is both the
`error-handling` plugin and the `cdc-errors` marketplace that lists it,
with `plugin.json` and `marketplace.json` carrying independent `version`
fields as required.

## Consequences

### Positive

1. **One repo to operate**: CI, branch protection, and release automation
   are configured once, not duplicated across a plugin repo and a
   marketplace repo.
2. **No cross-repo synchronization problem**: the catalog-pinning automation
   (ADR-0006) writes back within the same repo's permission boundary.

### Negative

1. **Local development testing is affected by SHA-pinning** (see ADR-0006):
   once the catalog entry is pinned to a released SHA rather than a live
   `"./"` reference, a locally-added marketplace no longer serves a
   developer's uncommitted local changes — it still resolves to the last
   pinned release. This is an accepted tradeoff, not an oversight.

### Neutral

1. **If this maintainer later wants a second, genuinely external plugin in
   this catalog**, the schema already supports mixing a local self-hosted
   entry with external `github`/`git-subdir` entries in one
   `marketplace.json` — no restructuring required, just an additional
   entry.

## Decision Outcome

Measured by: `/plugin marketplace add zircote/cdc-error-plugin` followed by
`/plugin install error-handling@cdc-errors` succeeds from a clean Claude
Code install, and `claude plugin validate . --strict` passes on both
manifests — both confirmed working during this session.

## Related Decisions

- [ADR-0006: Self-Referential Catalog SHA-Pinning via PAT-Authenticated Automation](0006-self-referential-catalog-pinning.md) - depends on this repo's single-repo shape; a separate marketplace repo would have required a cross-repo version of the same mechanism.

## Links

- [Claude Code: Create and distribute a plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) - the official schema this decision relies on, fetched and verified during this session rather than assumed from training data.

## More Information

- **Date:** 2026-07-03
- **Source:** explicit user requirement ("This marketplace will serve for now only the one plugin it holds and will be versioned at both the plugin and the marketplace level").
- **Related ADRs:** 0006

## Audit

### 2026-07-03

**Status:** Compliant

**Findings:**

| Finding | Files | Lines | Assessment |
| --- | --- | --- | --- |
| marketplace.json and plugin.json both present with independent version fields | `.claude-plugin/marketplace.json`, `.claude-plugin/plugin.json` | - | accepted |

**Summary:** Single-repo dual-role architecture implemented and verified via
`claude plugin validate . --strict`.

**Action Required:** None.

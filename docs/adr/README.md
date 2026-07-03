---
id: adr-index
type: semantic
created: 2026-07-03T00:00:00-04:00
diataxis_type: reference
title: Architectural Decision Records
---

# Architectural Decision Records

Structured MADR-format records of every architecturally significant,
hard-to-reverse decision this repo makes. See
[ADR-0001](0001-adopt-architectural-decision-records.md) for why this
practice exists and what warrants an ADR versus a plain `CHANGELOG.md` entry.

| ADR | Title | Status |
| --- | --- | --- |
| [0001](0001-adopt-architectural-decision-records.md) | Adopt Architectural Decision Records for This Project | Accepted |
| [0002](0002-single-repo-plugin-and-marketplace.md) | Single-Repo Plugin-and-Marketplace Architecture | Accepted |
| [0003](0003-full-tier-attestation-scope.md) | Full-Tier Attestation Scope for the Release Pipeline | Accepted |
| [0004](0004-no-central-attestation-signer.md) | Self-Signed Attestations, No Central Signer | Accepted |
| [0005](0005-default-token-for-release-auth.md) | Default GITHUB_TOKEN for Release Publishing | Accepted |
| [0006](0006-self-referential-catalog-pinning.md) | Self-Referential Catalog SHA-Pinning via PAT-Authenticated Automation | Accepted |
| [0007](0007-github-flow-branch-model.md) | GitHub Flow Branch Model | Accepted |

## Lifecycle

```text
proposed -> accepted -> deprecated
                   \-> superseded (by a newer ADR)
```

An accepted ADR is immutable: a change of mind gets a new ADR that
supersedes the old one, not an edit to the original's Decision or
Consequences sections.

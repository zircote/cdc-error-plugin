---
id: docs-index
type: semantic
created: 2026-07-03T00:00:00-04:00
diataxis_type: reference
title: Documentation index
---

# Documentation index

This documentation is organized per [Diátaxis](https://diataxis.fr/): four
kinds of content, each answering a different question. Pick the section
that matches what you're trying to do.

## Tutorial — learning by doing

Start here if you're new to the plugin.

- [Get started with the error-handling plugin](tutorials/getting-started.md) — install the plugin and watch `cdc-review` and `cdc-err` trigger on a real piece of code, start to finish.

## How-to guides — accomplishing a specific task

You already know what you want to do; these are the steps.

- [Install the plugin](how-to/install.md)
- [Add cdc-err dual-format output to an existing CLI](how-to/add-cdc-err-to-cli.md)
- [Run and improve the skill evals](how-to/run-evals.md)
- [Verify a release](how-to/verify-release.md)

## Reference — information-oriented lookup

Dry, exhaustive material you consult, not read start to finish.

- [Reference index](reference/index.md) — envelope schema, severity taxonomy, language-specific patterns, plugin/marketplace manifest fields, trigger phrases.

## Explanation — understanding the why

Background, rationale, and design decisions.

- [Why CLI errors are a dual-consumer problem](explanation/dual-consumer.md) — the framing `cdc-err` is built on.
- [Why three sibling skills, not one](explanation/skill-cooperation.md) — why the plugin splits producer / propagation / consumer across `cdc-err`, `cdc-review`, and `cdc-handle`.
- [Why this plugin ships an attested release pipeline](explanation/attested-releases.md) — why fail-closed release verification is worth building, and why it's scoped the way it is.

## Root-level documents

Outside `docs/`, at the repository root:

- [README.md](../README.md) — project overview, installation quick-start, plugin layout.
- [CHANGELOG.md](../CHANGELOG.md) — Keep a Changelog history.
- [SECURITY.md](../SECURITY.md) — vulnerability reporting, release verification model.

---
diataxis_type: explanation
---

# error-handling

A Claude Code plugin with three sibling skills covering the full error-handling lifecycle — producer, propagation, and consumer:

- **`cdc-err`** *(producer)* — designs, reviews, or refactors CLI error *output* using the [RFC 9457 Problem Details](https://www.rfc-editor.org/rfc/rfc9457) envelope and the dual-consumer pattern from [*"CLI Error Messages Are a Dual-Consumer Problem"*](https://zircote.com/blog/2026/04/cli-error-messages-are-a-dual-consumer-problem/).
- **`cdc-review`** *(propagation)* — reviews source-code error-handling quality (propagation, swallowing, cause-chain preservation, panic discipline, resource cleanup) across any language and any context, with classified findings, refactor diffs, and YAML work plans for large efforts.
- **`cdc-handle`** *(consumer)* — teaches an LLM agent how to interpret and act on a `cdc-err`-shaped envelope when it lands in a `tool_result`: parse, decide (retry / apply-fix / handback / abort), and emit an executable action plan gated on applicability markers.

The skills are designed to cooperate. Each declines cleanly outside its lane and hands off to the right sibling: `cdc-err` defers code questions to `cdc-review`; `cdc-review` defers envelope questions to `cdc-err`; `cdc-handle` defers producer questions to `cdc-err` and propagation questions to `cdc-review`.

## The problem this solves

Your CLI now has two audiences:

| | Human at terminal | LLM agent running the tool |
|---|---|---|
| Wants | Rich, colorized prose | Structured fields it can branch on |
| Reads | `stderr` directly | `tool_result` payload |
| Costs of getting it wrong | Frustration | Wasted tokens, abandoned recoverable work, wrong fixes auto-applied |

Most CLIs do (1) well and (2) badly. A 5KB traceback in a `tool_result` is ~1,600 tokens; a rate-limit error without `retry_after` makes Claude abandon the task entirely; an unmarked "suggested fix" gets applied verbatim even when it's wrong for your platform.

And the surface beneath the CLI — your library code, your handlers, your retry loops — often has its own pile of error-handling debt: swallowed exceptions, `unwrap()` in library code, lost cause chains, log-and-rethrow patterns producing duplicate operator alerts. And the *agent* on the other end of the CLI must decide whether to retry, apply a fix, or hand the situation back to a human — without re-inventing the contract the producer already wrote. The three skills together address all three surfaces.

## The three skills at a glance

### `cdc-err` — CLI error output

Triggers on phrases like *"review my CLI error messages"*, *"my CLI dumps a traceback when the API rate-limits us"*, *"add a compliant error to my Cobra CLI"*, *"problem+json"*, *"dual-consumer errors"*, *"RFC 9457"*.

Produces:
1. **Human rendering** — pretty `stderr` for TTY (Rich / miette / Cobra prose, preserved from your existing output).
2. **JSON envelope** — `application/problem+json` with all five RFC 9457 standard fields plus the three mandatory agent extensions (`retry_after`, `suggested_fix`, `code_actions[]`) and rustc-style applicability markers (`machine_applicable` / `maybe_incorrect` / `has_placeholders` / `unspecified`).
3. **Renderer-selection logic** — `--format=json|pretty` flag + TTY auto-detection, in your language.
4. **Cost justification** — token math for selling the migration to a teammate.

Three modes: **Design** (start from scratch), **Review** (audit existing output), **Refactor** (convert specific message to dual format).

### `cdc-review` — source-code error handling

Triggers on phrases like *"review my error handling"*, *"audit error code"*, *"find error-handling bugs"*, *"review my Result usage"*, *"audit my errors.Is calls"*, *"check my try/except"*, *"panic in library"*, *"swallowed error"*, *"lost cause chain"*. Also auto-triggers when a PR / file edit touches error-emission, propagation, recovery, or cleanup code.

Produces (one of):
1. **Classified violation report** — `[must-fix]` / `[should-fix]` / `[nit]` / `[question]` / `[praise]` findings with `what` / `why` / `fix` for each. Always includes ≥1 praise if anything is done well.
2. **Concrete before/after diffs** — for must-fix and should-fix findings, ready for Claude to apply via Edit.
3. **YAML work plan** — for large refactors (>5 findings or >3 files), per-finding issue specs (problem, expected, acceptance criteria, effort, dependencies, labels, priority, milestone). Hands off to `/gh-work` for actual issue creation.
4. **Severity classification** — uses the canonical Anthropic `pr-review` taxonomy, specialized to error-handling criteria.

Four modes: **Review**, **Refactor**, **Design** (when no code exists yet), **Work-Plan** (large effort).

### `cdc-handle` — agent-side envelope interpretation and action

Triggers on phrases like *"interpret this error"*, *"should I retry this CLI error"*, *"the tool_result has problem+json"*, *"can I auto-apply this suggested_fix"*, *"what does this retry_after mean"*. Also auto-triggers whenever a `tool_result` payload contains the cdc-err contract fields (`retry_after` / `suggested_fix` / `code_actions[]`) or a `problem+json` body.

Produces an **executable action plan** in one of three modes:
1. **Parse** — structured summary of the envelope (envelope_kind, status_class, extension values, applicability markers, malformations).
2. **Decide** — a single decision (`retry` / `apply_fix` / `handback` / `abort`) with the reasoning and the envelope fields the decision rests on.
3. **Act** — a JSON action plan the agent can execute, with each step carrying `kind`, `applicability_gate`, `action`, `args`, and `on_failure`.

The non-negotiable is the applicability gate: a `suggested_fix` or `code_actions[]` entry is only auto-applied when its applicability is `machine_applicable`. Anything else — `maybe_incorrect`, `has_placeholders`, `unspecified`, or absent applicability — routes to `handback`. This is the consumer half of the contract `cdc-err` writes on the producer side.

## How the skills cooperate

| Situation | Which skill |
|---|---|
| "Review my Rust CLI's error propagation chain" | `cdc-review` |
| "What fields should my problem+json envelope include?" | `cdc-err` |
| "I just got back a 429 with retry_after=30 — should I retry?" | `cdc-handle` |
| "Review this Cobra command — propagation **and** output format" | Both. `cdc-review` covers propagation; `cdc-err` covers output. They cross-link. |
| "The CLI returned a problem+json error — what should the agent do?" | `cdc-handle` |
| "My Python library swallows database errors silently" | `cdc-review` |
| "Convert my CLI's rate-limit error to dual format" | `cdc-err` |
| "Can I auto-apply this suggested_fix that came back from the CLI?" | `cdc-handle` |
| "Refactor my service's exception hierarchy" | `cdc-review` |

When any skill encounters work outside its scope, it names the right sibling and hands off rather than inventing guidance outside its lane.

## Documentation

Organized per [Diátaxis](https://diataxis.fr/) under [`docs/`](docs/):

- **Tutorial** — [Get started with the plugin](docs/tutorials/getting-started.md)
- **How-to** — [Install](docs/how-to/install.md) · [Add cdc-err to a CLI](docs/how-to/add-cdc-err-to-cli.md) · [Run evals](docs/how-to/run-evals.md) · [Verify a release](docs/how-to/verify-release.md)
- **Reference** — [Index](docs/reference/index.md) (envelope, severity, language refs)
- **Explanation** — [Dual-consumer problem](docs/explanation/dual-consumer.md) · [Why two sibling skills](docs/explanation/skill-cooperation.md)

## Installation

This repository doubles as its own single-plugin marketplace
(`.claude-plugin/marketplace.json` lists the one plugin, sourced from the
repo root):

```bash
/plugin marketplace add zircote/cdc-error-handling
/plugin install error-handling@error-handling
```

For local development, point Claude Code at a checkout directly:

```bash
/plugin marketplace add /absolute/path/to/this/repo
/plugin install error-handling@error-handling
```

All three skills auto-discover from `skills/`. No further configuration.
See [docs/how-to/install.md](docs/how-to/install.md) for details.

## Versioning and releases

The plugin (`.claude-plugin/plugin.json`) and the marketplace catalog
(`.claude-plugin/marketplace.json`) carry independent semver `version`
fields — installing users track the plugin version; the catalog version
tracks changes to the marketplace listing itself. Plugin changes are
recorded in [CHANGELOG.md](CHANGELOG.md).

Tagged releases (`vX.Y.Z`) build a reproducible source tarball, attest its
SLSA build provenance, and fail-closed-verify that attestation before
publishing — see [SECURITY.md](SECURITY.md) for the security model and
[docs/how-to/verify-release.md](docs/how-to/verify-release.md) for how to
independently verify a downloaded release.

## When the skills won't trigger (and shouldn't)

Both skills decline cleanly outside their scope:

`cdc-err` declines on:
- REST/HTTP API errors consumed by a browser frontend (use RFC 9457 directly; the "dual consumer" framing is CLI-specific).
- Application logging (structlog/slog/etc.) — different concern.
- Frontend exception display.
- Internal telemetry shape.

`cdc-review` declines on:
- Formatting, naming, performance, generic code style (use `pr-review` / `code-reviewer`).
- Test coverage of error paths (call out `/test-architect`).
- Security beyond error-related (SQL injection, XSS, auth — different domain). Information leakage *in* error messages **is** in scope.
- Business-logic correctness.

`cdc-handle` declines on:
- Non-RFC-9457 payloads (raw stderr, ad-hoc JSON error shapes, plaintext tracebacks). Defer to a general-purpose error reader.
- Designing the envelope — that's `cdc-err`.
- Reviewing the CLI's internal error-propagation code — that's `cdc-review`.
- Generic HTTP retry policies for browser frontends.

If you ask outside their scope, they say so and use general knowledge or point you at the right tool.

## What's out of scope (honestly marked)

`cdc-err` is faithful to its source post; where the post is silent, the skill says so:

- **Complete HTTP-status → exit-code table.** The post commits to `429 → 2`; the rest is the CLI author's call.
- **Localization** of `title` / `detail` strings.
- **Telemetry/logging** of emitted errors.
- **Idiomatic libraries for Node/Ruby/Java/C#/shell.** The pattern applies; specific library choices are yours.
- **Migration path from existing error taxonomies** to stable `type` URIs.

`cdc-review` draws on multiple sources (Rust API guidelines, Go errors design, PEP 3134, Effective Java), cited per-principle. Where principles are contested or language-specific, the relevant `references/languages/<lang>.md` names the source rather than claiming universal authority.

## Plugin layout

```
.
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json              # this repo doubles as its own marketplace
├── .github/
│   └── workflows/
│       ├── ci.yml                     # pin-check, actionlint, claude plugin validate
│       └── release.yml                # tarball -> attest-build-provenance -> verify -> publish
├── skills/
│   ├── cdc-err/
│   │   ├── SKILL.md
│   │   ├── references/
│   │   │   ├── envelope.md          # RFC 9457 spec + 3 mandatory extensions + applicability markers
│   │   │   ├── languages.md         # Rust/miette, Python/Rich, Go/Cobra
│   │   │   └── review-checklist.md  # 7-section audit checklist for error output
│   │   └── evals/
│   │       └── evals.json           # 12 evals, 50 LLM expectations, 63 deterministic checks (55.8%)
│   ├── cdc-review/
│   │   ├── SKILL.md
│   │   ├── references/
│   │   │   ├── checklist.md         # 8-section code review checklist
│   │   │   ├── severity.md          # must-fix/should-fix/nit/question/praise + error-handling criteria
│   │   │   ├── workplan.md          # YAML work-plan format for large refactors
│   │   │   └── languages/
│   │   │       ├── rust.md          # thiserror/anyhow, ?, panic boundaries
│   │   │       ├── go.md            # errors.Is/As, %w, errors.Join
│   │   │       ├── python.md        # PEP 3134, raise from, with, contextlib.suppress
│   │   │       ├── typescript.md    # Error cause, neverthrow, async pitfalls
│   │   │       └── java.md          # try-with-resources, checked vs unchecked
│   │   └── evals/
│   │       └── evals.json           # 12 evals, 53 expectations, 53 deterministic checks (50%)
│   └── cdc-handle/
│       ├── SKILL.md
│       ├── references/
│       │   ├── parsing.md           # recognize cdc-err envelopes; status classes; malformation catalogue
│       │   ├── decision-tree.md     # retry / apply_fix / handback / abort lattice
│       │   ├── applicability.md     # machine_applicable / maybe_incorrect / has_placeholders / unspecified gate
│       │   └── code-actions.md      # translating code_actions[] entries into executable plan steps
│       └── evals/
│           └── evals.json           # 15 evals, 69 expectations, 76 deterministic checks (52.4%)
├── README.md
├── CHANGELOG.md                      # Keep a Changelog format
├── SECURITY.md                       # verification model + gh attestation verify commands
├── docs/                             # Diátaxis-organized docs (tutorials/how-to/reference/explanation)
└── .gitignore                        # see file for full ignore list
```

Standard Claude Code plugin layout — `.claude-plugin/plugin.json` at the root, skills auto-discovered under `skills/`. No `skills` array in `plugin.json` (auto-discovery handles that).

## Iterating on either skill

The repo is set up for autonomous skill improvement via the [`autoresearch`](https://github.com/zircote/autoresearch) plugin:

```bash
# Audit/upgrade the eval set (deterministic checks, discriminating expectations)
/autoresearch --eval-doctor skills/cdc-err
/autoresearch --eval-doctor skills/cdc-review
/autoresearch --eval-doctor skills/cdc-handle

# Run the modify → evaluate → keep-or-discard loop
/autoresearch skills/cdc-err
/autoresearch skills/cdc-review
/autoresearch skills/cdc-handle
```

Both eval sets carry ≥50% deterministic-check ratio with grep-anchored regex on the canonical RFC 9457 fields (`cdc-err`) and the error-handling pattern tokens (`cdc-review`) — so the loop has real signal to optimize against, not just presence-only assertions.

## Validation

Canonical validation, run on every push/PR in `.github/workflows/ci.yml`:

```bash
claude plugin validate . --strict
```

This checks `marketplace.json` and `plugin.json` schema conformance, and
(for the local `./` plugin entry) `plugin.json` and each skill's SKILL.md
frontmatter. CI also runs `actionlint` against the workflow files and
`pin-check` (every `uses:` in `.github/workflows` must be a full 40-char
commit SHA).

## Sources and authority

- **`cdc-err`** traces all prescriptions to https://zircote.com/blog/2026/04/cli-error-messages-are-a-dual-consumer-problem/. The post is the only authority; anything outside it is marked **out of scope** rather than invented. Upstream sources cited in the post: RFC 9457, RFC 7231 §7.1.3, SARIF 2.1.0, LSP 3.17, Anthropic tool-use docs, miette, rustc diagnostic guide.
- **`cdc-review`** draws on multiple sources: Rust API guidelines (`C-GOOD-ERR`, `C-PANIC-FREE`), Go `errors` package guidance + 2019 error-values design, PEP 3134 (Python exception chaining), *Effective Java* items 69-77, ES2022 `Error.cause`. Each principle is cited in the relevant language reference rather than at a single canonical URL.
- **`cdc-handle`** consumes the same contract `cdc-err` produces. Authority chain: cdc-err's `references/envelope.md` is the schema; RFC 9457 § 3 (Problem Details for HTTP APIs) is the upstream standard; rustc's diagnostic guide is the source of the applicability markers (`machine_applicable` / `maybe_incorrect` / `has_placeholders` / `unspecified`); LSP's `CodeAction` interface is the shape used for `code_actions[]` entries.

## License

MIT

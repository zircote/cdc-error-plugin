# error-handling

A Claude Code plugin with two sibling skills for the full error-handling lifecycle:

- **`cdc-err`** — designs, reviews, or refactors CLI error *output* using the [RFC 9457 Problem Details](https://www.rfc-editor.org/rfc/rfc9457) envelope and the dual-consumer pattern from [*"CLI Error Messages Are a Dual-Consumer Problem"*](https://zircote.com/blog/2026/04/cli-error-messages-are-a-dual-consumer-problem/).
- **`cdc-review`** — reviews source-code error-handling quality (propagation, swallowing, cause-chain preservation, panic discipline, resource cleanup) across any language and any context, with classified findings, refactor diffs, and YAML work plans for large efforts.

The skills are designed to cooperate: `cdc-review` defers all envelope/format questions to `cdc-err`, and `cdc-err` defers all code-propagation questions to `cdc-review`. Either skill will name and hand off to the other when the work crosses the boundary.

## The problem this solves

Your CLI now has two audiences:

| | Human at terminal | LLM agent running the tool |
|---|---|---|
| Wants | Rich, colorized prose | Structured fields it can branch on |
| Reads | `stderr` directly | `tool_result` payload |
| Costs of getting it wrong | Frustration | Wasted tokens, abandoned recoverable work, wrong fixes auto-applied |

Most CLIs do (1) well and (2) badly. A 5KB traceback in a `tool_result` is ~1,600 tokens; a rate-limit error without `retry_after` makes Claude abandon the task entirely; an unmarked "suggested fix" gets applied verbatim even when it's wrong for your platform.

And the surface beneath the CLI — your library code, your handlers, your retry loops — often has its own pile of error-handling debt: swallowed exceptions, `unwrap()` in library code, lost cause chains, log-and-rethrow patterns producing duplicate operator alerts. The two skills together address both surfaces.

## The two skills at a glance

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

## How the skills cooperate

| Situation | Which skill |
|---|---|
| "Review my Rust CLI's error propagation chain" | `cdc-review` |
| "What fields should my problem+json envelope include?" | `cdc-err` |
| "Review this Cobra command — propagation **and** output format" | Both. `cdc-review` covers propagation; `cdc-err` covers output. They cross-link. |
| "My Python library swallows database errors silently" | `cdc-review` |
| "Convert my CLI's rate-limit error to dual format" | `cdc-err` |
| "Refactor my service's exception hierarchy" | `cdc-review` |

When either skill encounters work outside its scope, it names the sibling and hands off rather than inventing guidance outside its lane.

## Installation

Add this plugin to a Claude Code marketplace, then enable it:

```bash
# If you maintain a personal marketplace pointing at this repo
claude plugins add error-handling

# Or, for local development, point Claude Code at this directory directly
# via .claude/settings.json:
# {
#   "plugins": {
#     "marketplaces": [{ "path": "/absolute/path/to/this/repo" }]
#   }
# }
```

Both skills are auto-discovered from `skills/`. No further configuration.

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
│   └── plugin.json
├── skills/
│   ├── cdc-err/
│   │   ├── SKILL.md
│   │   ├── references/
│   │   │   ├── envelope.md          # RFC 9457 spec + 3 mandatory extensions + applicability markers
│   │   │   ├── languages.md         # Rust/miette, Python/Rich, Go/Cobra
│   │   │   └── review-checklist.md  # 7-section audit checklist for error output
│   │   └── evals/
│   │       └── evals.json           # 12 evals, 50 LLM expectations, 63 deterministic checks (55.8%)
│   └── cdc-review/
│       ├── SKILL.md
│       ├── references/
│       │   ├── checklist.md         # 8-section code review checklist
│       │   ├── severity.md          # must-fix/should-fix/nit/question/praise + error-handling criteria
│       │   ├── workplan.md          # YAML work-plan format for large refactors
│       │   └── languages/
│       │       ├── rust.md          # thiserror/anyhow, ?, panic boundaries
│       │       ├── go.md            # errors.Is/As, %w, errors.Join
│       │       ├── python.md        # PEP 3134, raise from, with, contextlib.suppress
│       │       ├── typescript.md    # Error cause, neverthrow, async pitfalls
│       │       └── java.md          # try-with-resources, checked vs unchecked
│       └── evals/
│           └── evals.json           # 12 evals, 53 expectations, 53 deterministic checks (50%)
├── README.md
└── .gitignore                       # ignores transient *-autoresearch/, *-workspace/, *-dashboard.html
```

Standard Claude Code plugin layout — `.claude-plugin/plugin.json` at the root, skills auto-discovered under `skills/`. No `skills` array in `plugin.json` (auto-discovery handles that).

## Iterating on either skill

The repo is set up for autonomous skill improvement via the [`autoresearch`](https://github.com/zircote/marketplace) plugin:

```bash
# Audit/upgrade the eval set (deterministic checks, discriminating expectations)
/autoresearch --eval-doctor skills/cdc-err
/autoresearch --eval-doctor skills/cdc-review

# Run the modify → evaluate → keep-or-discard loop
/autoresearch skills/cdc-err
/autoresearch skills/cdc-review
```

Both eval sets carry ≥50% deterministic-check ratio with grep-anchored regex on the canonical RFC 9457 fields (`cdc-err`) and the error-handling pattern tokens (`cdc-review`) — so the loop has real signal to optimize against, not just presence-only assertions.

## Validation

Both skills pass the canonical Anthropic skill validator:

```bash
python3 -c "from scripts.quick_validate import validate_skill; \
  print(validate_skill('./skills/cdc-err'))"
# (True, 'Skill is valid!')

python3 -c "from scripts.quick_validate import validate_skill; \
  print(validate_skill('./skills/cdc-review'))"
# (True, 'Skill is valid!')
```

`plugin.json` follows the canonical Claude Code plugin schema (`name`, `version`, `description`, `author`, `license`, `keywords`).

## Sources and authority

- **`cdc-err`** traces all prescriptions to https://zircote.com/blog/2026/04/cli-error-messages-are-a-dual-consumer-problem/. The post is the only authority; anything outside it is marked **out of scope** rather than invented. Upstream sources cited in the post: RFC 9457, RFC 7231 §7.1.3, SARIF 2.1.0, LSP 3.17, Anthropic tool-use docs, miette, rustc diagnostic guide.
- **`cdc-review`** draws on multiple sources: Rust API guidelines (`C-GOOD-ERR`, `C-PANIC-FREE`), Go `errors` package guidance + 2019 error-values design, PEP 3134 (Python exception chaining), *Effective Java* items 69-77, ES2022 `Error.cause`. Each principle is cited in the relevant language reference rather than at a single canonical URL.

## License

MIT

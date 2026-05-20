---
name: cdc-err
description: Design, review, or refactor CLI error messages so they serve BOTH human operators AND LLM agents using the RFC 9457 Problem Details envelope with mandatory agent extensions (retry_after, suggested_fix, code_actions[]). Use whenever the user works on CLI error output, stderr format, exit codes, machine-readable errors, structured error envelopes, dual-format CLI rendering, or reducing token cost of tool_result payloads. Triggers on phrases like "CLI error message", "stderr format", "machine-readable errors", "dual-consumer errors", "problem+json", "RFC 9457 errors", "agent-parseable errors", "review my CLI error messages", "refactor error handling", or when improving error reporting in any CLI (Go, Rust, Python, Node, Ruby, Java, C#, shell). Use this skill even when the user does not cite the RFC or dual-consumer framing — if the work touches CLI error output an agent might parse, this skill applies.
argument-hint: "[design|review|refactor] [code-or-error-output-or-language]"
allowed-tools: "Read, Edit, Write, Grep, Glob, Bash"
license: MIT
---

# CLI Error Messages as a Dual-Consumer Problem

This skill codifies the prescriptions from Allen Walker's post *"CLI Error Messages Are a Dual-Consumer Problem"* (https://zircote.com/blog/2026/04/cli-error-messages-are-a-dual-consumer-problem/). Every rule below traces to that post. Anything outside it is marked **(out of scope for this skill)**.

## The Thesis (verbatim premise)

Command-line tools now answer to **two distinct audiences**:

1. **The human** who ran the command and reads the terminal.
2. **The LLM agent** that orchestrated the command, parses the bytes, and decides whether to retry, escalate, or abandon.

> Most CLI error messages serve the first audience adequately and the second audience poorly. That imbalance is not a polish problem. It is a cost problem, a reliability problem, and a convergence problem.

A CLI that ignores the second consumer:
- Burns tokens on verbose tracebacks (a 5KB traceback ≈ 1,600 tokens of tool_result; structured JSON can cut this ~75%).
- Causes agents to abandon recoverable tasks (e.g. rate-limit errors without `Retry-After`).
- Makes agents apply plausible-looking but wrong fixes when applicability is ambiguous.

## What This Skill Does

Pick one of three modes based on what the caller provided. The decision rules are deterministic — only ask if the input is genuinely ambiguous (e.g. *"help me with my errors"* with no further context):

| Mode | Decision rule | Output shape |
|---|---|---|
| **Design** | Caller asks *how should I…* / *what should I emit when…* / *what do I commit to before release…* — no concrete error in hand. | Compliant envelope (all five RFC 9457 fields + three mandatory extensions), dual-renderer plan for caller's language, release-level commitments where relevant. |
| **Review** | Caller pastes existing error output (stderr lines, log excerpts, or a partial JSON envelope) and asks for a review / audit / "what's missing". | One-by-one violations using `VIOLATION / EVIDENCE / POST CITATION / FIX` (see `references/review-checklist.md`). Quote the post line each violation breaks. |
| **Refactor** | Caller has a specific error message and asks to convert / fix / make agent-friendly. | Side-by-side: the human rendering (preserved, *"as lush as it ever was"*) and the `application/problem+json` envelope. Renderer-selection logic (`--format` flag + TTY) in the caller's language. |

Once the mode is clear, load only the reference file you need:

| File | Load when… |
|---|---|
| `references/envelope.md` | You need exact field semantics, applicability marker definitions, or versioning policy (≈ every Design and Refactor request). |
| `references/languages.md` | Caller named a language. Load **only** the section for that language — Rust (miette + thiserror), Python (Rich + Click/Typer), Go (Cobra + `encoding/json`). |
| `references/review-checklist.md` | Review mode — walk the 7 sections in order. |

### What every output must include

Regardless of mode, the output is incomplete unless it covers:

1. The five RFC 9457 standard fields (`type`, `title`, `status`, `detail`, `instance`).
2. The three mandatory agent extensions (`retry_after`, `suggested_fix`, `code_actions[]`) — with `retry_after` explicitly set (even `0`/`null`) on non-transient errors so agents don't have to guess.
3. Applicability marker on every `suggested_fix` and every `code_actions[]` entry.
4. The dual-renderer selection mechanism (`--format` and TTY).
5. A stable `type` URI — versioned in path (`/v1/…`) or stable with a documentation changelog.

If any of these is genuinely out of scope for the request (e.g. a Review of a 3-line stderr line doesn't need a full envelope), say so explicitly. Silent omission is the failure mode the skill exists to prevent.

### Sibling skill: `cdc-review`

This skill owns CLI error *output* (envelope, format, rendering). For source-code-level error-handling review — propagation, swallowing, lost cause chains, panic discipline, resource cleanup on the error path, log-and-rethrow — defer to the sibling skill `cdc-review`. When a caller asks both "is my output right?" and "is my error-handling code right?", route the first half here and the second half to `cdc-review`.

## Non-Negotiables (from the post)

A compliant CLI error MUST:

1. Ship a **dual rendering** in the same binary. Selection is by explicit flag (`--format=json` / `--format=pretty`) **or** TTY detection. The human rendering stays "as lush as it ever was"; the agent gets the structured version.
2. Use the **RFC 9457 Problem Details** JSON envelope under media type `application/problem+json`. Standard members: `type`, `title`, `status`, `detail`, `instance`.
3. Include the **three mandatory extensions** for agent consumers:
   - `retry_after` — delta-seconds or timestamp indicating when the operation may safely be retried.
   - `suggested_fix` — free-text or structured description of the recovery action.
   - `code_actions[]` — structured edits modeled on the LSP `CodeAction` interface.
4. Carry **applicability markers** on suggested fixes (per rustc precedent): `machine_applicable`, `maybe_incorrect`, `has_placeholders`, `unspecified`. Without these, agents will apply wrong fixes.
5. Commit to a **stable `type` URI policy** — either the URI embeds a version, or the problem-type documentation tracks a changelog.

Optional but encouraged extensions: `exit_code`, `libraries_missing`, `docs_url`.

## Canonical Example (from the post)

Rate-limit error with the mandatory retry directive:

```json
{
  "type": "https://api.example.com/errors/rate-limit-exceeded",
  "title": "Rate limit exceeded",
  "status": 429,
  "detail": "Exceeded rate limit for this endpoint.",
  "instance": "urn:request:2026-04-15T14:22:10Z-req-abc123",
  "exit_code": 2,
  "retry_after": 180,
  "suggested_fix": "Wait 180 seconds before retrying.",
  "docs_url": "https://api.example.com/docs/rate-limits"
}
```

Without `retry_after`, "some model responses will abandon the task entirely." This is the failure mode the skill exists to prevent.

## Anti-Patterns to Flag (verbatim from the post)

When reviewing existing output, flag any of these:

- **Verbose tracebacks shipped in tool_result blocks.** Quantify the token cost; offer the structured equivalent.
- **Rate-limit / transient errors without retry directives.** Agents abandon recoverable work.
- **Unstructured error hints.** "An agent sees the same string... and cannot act on it because the hint omits the one thing the agent needs, which is a number."
- **Applicability ambiguity in suggested fixes.** Without `machine_applicable` / `maybe_incorrect` / etc., agents apply plausible-looking but wrong edits.
- **Single-format output.** Pretty-only or JSON-only both fail one audience.

## Implementation Checklist (verbatim, three composable components)

1. Emit errors as RFC 9457 JSON on `--format=json` or non-TTY.
2. Ship a dual renderer (human-readable terminal + machine-readable JSON).
3. Add the three extension members (`retry_after`, `suggested_fix`, `docs_url`) plus optional `code_actions[]`.

## Cost Model (use to justify the work)

```
cost_per_retry = (tool_floor + tool_result_bytes / bytes_per_token) * input_price_per_token
tool_floor      = 313–346 for Claude 4.x tool use
bytes_per_token ≈ 4 (conservative for English text)
```

The post claims ~75% per-tool_result reduction by replacing prose tracebacks with structured envelopes; over fifteen retries the savings compound to tens of thousands of tokens. Cite these numbers when callers ask whether the migration is worth it.

## Out of Scope for This Skill

The post is silent on the following. Do not invent guidance:

- Specific HTTP status-code → exit-code tables beyond the `status: 429 → exit_code: 2` example shown.
- Localization of `title` / `detail` strings.
- Telemetry or logging of emitted errors.
- Schema enforcement of extension members beyond their names and intent.
- How to migrate an existing tool's error taxonomy to stable `type` URIs (the post requires a versioning policy but does not prescribe one).

If a caller asks about any of the above, say so explicitly rather than guessing.

## Source

All prescriptions trace to: https://zircote.com/blog/2026/04/cli-error-messages-are-a-dual-consumer-problem/

The post cites RFC 9457, RFC 7231 §7.1.3, SARIF 2.1.0, LSP 3.17, Anthropic tool-use docs, miette, and the rustc diagnostic guide as upstream sources. Refer callers to those for material not covered here.

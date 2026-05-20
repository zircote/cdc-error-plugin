# Review Checklist for Existing CLI Error Output

Use this when a caller asks to audit or refactor existing error messages. Apply each item in order; report violations with the specific quote from the post that the violation breaks.

Source: https://zircote.com/blog/2026/04/cli-error-messages-are-a-dual-consumer-problem/

## 1. Dual Rendering Present?

- [ ] Does the CLI offer a `--format=json` (or equivalent) flag?
- [ ] Does it auto-select JSON when stdout/stderr is not a TTY?
- [ ] Is the human rendering preserved (not stripped to make room for JSON)?

**Violation if any answer is no.** The post: a dual-format CLI "ships both renderings in the same binary."

## 2. RFC 9457 Envelope Used for the JSON Path?

- [ ] Media type advertised as `application/problem+json` (where a media type is meaningful — HTTP, files, manifests).
- [ ] Standard members present: `type`, `title`, `status`, `detail`, `instance`.
- [ ] `type` is a real URI (not `about:blank`, unless the error is intentionally generic).
- [ ] `title` is stable per `type`; `detail` varies per occurrence.

## 3. Three Mandatory Extensions Present?

- [ ] `retry_after` on every transient / rate-limit / capacity error.
- [ ] `suggested_fix` on every error class that has any recovery action.
- [ ] `code_actions[]` modeled on LSP `CodeAction` where applicable.

**Top failure mode:** rate-limit errors without `retry_after`. The post: "some model responses will abandon the task entirely."

## 4. Applicability Markers on Every Fix?

For each `suggested_fix` and each entry in `code_actions[]`:

- [ ] Carries one of `machine_applicable`, `maybe_incorrect`, `has_placeholders`, `unspecified`.

Without these, agents apply plausible-looking but wrong fixes.

## 5. Stable `type` URI Policy Documented?

- [ ] Either URI embeds a version (`/v2/`) **or** the docs page tracks a changelog.
- [ ] No silent semantic drift across releases.

## 6. Anti-Pattern Scan

Flag if you see any of these in current output:

- [ ] Verbose tracebacks dumped into stderr — quantify token cost (1 KB ≈ 250 tokens; the post quotes 5 KB ≈ 1,600 tokens of tool_result floor + payload).
- [ ] Hints like "see logs for details" or "try again" without a number or directive.
- [ ] Identical JSON and pretty output (defeats the dual-format point in either direction).
- [ ] Pretty output only (the typical default — fails the agent consumer).
- [ ] JSON output only (fails the human consumer; the post insists on dual).

## 7. Exit Code / Status Sanity

- [ ] Exit code present and consistent with the error class.
- [ ] `status` field present in the envelope; relationship to `exit_code` is documented somewhere stable.

The post gives one mapping (rate-limit `status: 429 → exit_code: 2`); the full table is the author's call.

## Report Format

When reporting back to the caller, structure findings as:

```
VIOLATION: <which checklist item>
EVIDENCE: <quote of current output or code>
POST CITATION: <the relevant line from the blog post>
FIX: <minimal change to comply>
```

Do not file a violation for anything the post is silent on. Mark those as advisory only.

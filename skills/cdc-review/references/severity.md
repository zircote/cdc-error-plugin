# Severity Taxonomy

Standard five-class severity taxonomy used across most code-review workflows. Same five classes, same blocking semantics. Criteria below are specialized to error-handling code.

## The five classes

| Class | Blocks approval? | When to use |
|---|---|---|
| **must-fix** | Yes | Bug, data corruption, security, broken contract. The change cannot land without this. |
| **should-fix** | No (recommended) | Quality regression, missing recommended pattern, maintainability hit. Land-blocked only if part of a larger Work-Plan. |
| **nit** | No (optional) | Style, naming, choice of equally-correct idiom. The reviewer would not lose sleep if ignored. |
| **question** | Info only | Reviewer doesn't know if the code is correct without more context. The author should answer, not necessarily change anything. |
| **praise** | Always include ≥1 | Acknowledge what was done well. Calibrates the rest of the review. |

## must-fix criteria (error-handling)

- **Silent swallow** of a fallible operation with no recovery and no log.
- **`panic!()` / `process.exit()` / `os.Exit()` in library code.** Process-abort from a library is not a library.
- **Lost cause chain** where the upstream error carried diagnostic info (the wrap dropped it).
- **Information leakage** — secret, internal path, raw SQL, PII, or stack trace in a user-visible error.
- **Missing cleanup on the error path** for a resource with system-wide impact (DB conn, file handle, network socket, mutex).
- **Type-erased `Error` / `interface{}` / `Box<dyn Error>` at API boundaries** where the caller has no way to discriminate failure modes that need different handling.

## should-fix criteria (error-handling)

- **Generic catch-all** (`except Exception`, `catch (Throwable)`, `catch (Exception e)` outside a top-level boundary).
- **Missing context wrap** — propagating a low-level error to a high-level boundary without naming the operation that failed.
- **Log-and-rethrow** producing duplicate log lines.
- **Panic in CLI `main` without recovery** — the binary crashes instead of emitting a proper error envelope.
- **Inconsistent error type within the same module** (mix of typed errors and raw strings).
- **Missing `errors.Is` / pattern-match capability** because the wrap used string formatting instead of `%w`.
- **Recoverable error returned as terminal** (or vice versa) — boundary classification wrong.

## nit criteria (error-handling)

- **Sentinel vs typed errors** where either works for the use case.
- **Error message string style** — capitalization, period at end (Go convention says no, others don't care).
- **Log level choice** for an expected-failure path (debug vs info).
- **Order of `errors.Is` / `errors.As` checks** when the order doesn't change semantics.

## question criteria

Use when the reviewer suspects something is wrong but can't tell without seeing more code:
- "Is this `unwrap` reachable when `parse` fails on user input?"
- "Did you mean to drop the error here, or is this a TODO?"
- "Is the caller expected to retry on this error class?"

Make questions specific. A vague "are you sure?" is noise.

## praise criteria

Always include ≥1 if any error handling is done well. Specific examples worth calling out:

- **Cause chain preserved across multiple wrap sites** consistently.
- **Typed error hierarchy that lets callers branch correctly** with `errors.As` / `match` / `instanceof`.
- **Resource cleanup correct on every path** including panic / exception unwind.
- **Boundary classification done deliberately** — different error categories handled differently at the boundary, with rationale.
- **Log-once-at-boundary discipline** with full cause chain.

Praise that reads as filler ("nice job") doesn't calibrate the review. Praise specific patterns.

## Severity inflation

Most reviews err toward marking everything must-fix. Two checks before promoting:

1. **Could this ship today and cause an outage?** If yes → must-fix. If "it would just be ugly" → should-fix.
2. **Does the existing code have many similar instances?** If yes → file one must-fix for the *pattern*, not one per instance. Use the Work-Plan mode (see `workplan.md`) to track the cleanup.

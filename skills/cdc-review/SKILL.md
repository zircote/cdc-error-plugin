---
name: cdc-review
description: Review, refactor, or design error-handling source code across any language (Rust, Go, Python, TypeScript, Java, others) and any context (CLI, library, service, web). Produces classified violation reports (must-fix / should-fix / nit / question / praise), before/after diffs, and YAML work plans for large refactors. Triggers on "review my error handling", "audit error code", "find error-handling bugs", "fix exception handling", "improve error propagation", "review my Result usage", "audit my errors.Is calls", "check my try/except", "panic in library", "swallowed error", "lost cause chain", or when reviewing code where modified files contain error emission, propagation, recovery, or cleanup logic. Defers CLI envelope and format questions to sibling skill cdc-err (RFC 9457 Problem Details). Use even when the user does not request a review — if the work touches how errors propagate, get swallowed, get wrapped, or get logged, this skill applies.
argument-hint: "[review|refactor|design|workplan] [code-snippet-or-path]"
allowed-tools: "Read, Edit, Write, Grep, Glob, Bash"
license: MIT
---

# Error-Handling Code Review

This skill reviews **source code** that emits, propagates, recovers from, or logs errors. It is language-agnostic and context-agnostic — CLI, library, service, frontend.

It does **not** dictate the output shape of CLI errors at the binary boundary — for that, the sibling skill `cdc-err` codifies the RFC 9457 + dual-consumer pattern. When a review crosses into envelope/format territory, defer to `cdc-err`.

## Authority and honest scope

This skill draws on established error-handling principles from multiple sources rather than a single canonical post:

- **Rust:** the Rust API guidelines (`C-GOOD-ERR`), plus the wider Rust community convention against panicking/aborting in library code — the latter is not a numbered guideline in the API guidelines checklist, and this skill doesn't claim it is.
- **Go:** `errors` package guidance, the 2019 error-values proposal, `errors.Is` / `errors.As` / `%w` semantics.
- **Python:** PEP 3134 (exception chaining), the `contextlib` docs, the cpython `Exception` hierarchy.
- **JVM:** Effective Java items 69-77 on exception design.
- **CLI output:** delegated to `cdc-err`.

Where principles are contested or language-specific, the relevant `references/languages/<lang>.md` file cites the source per-principle rather than claiming universal authority.

## What this skill produces

Pick mode from the input shape. The four modes are mutually exclusive — only one runs.

| Input shape | Mode | Output |
|---|---|---|
| User pastes code and asks to review/audit/check | **Review** | Classified violation report (must-fix / should-fix / nit / question / praise), one finding per row with file:line, what/why/suggestion. |
| User pastes code and asks to fix/refactor/improve | **Refactor** | Before/after diffs for each must-fix and should-fix, ready for Edit. Findings classified the same way. |
| User asks "how should I handle errors for X" with no code in hand | **Design** | Idiomatic patterns + anti-patterns for the language and context. Cross-link to `cdc-err` if X is a CLI. |
| Review finds >5 findings OR spans >3 files | **Work-Plan** | YAML decomposition per `references/workplan.md`. Each finding becomes an issue spec (title, problem, expected, acceptance criteria, effort, dependencies, labels, priority). The YAML is the deliverable — issue creation, if desired, is the user's choice using whatever tool their workflow uses. |

If a review request looks large up front (>500 lines OR >10 files), surface the recommendation to parallelize: spawn specialist agents (one per language or layer) so the review completes in reasonable time. This skill names the trigger condition; the user (or a parent agent) decides how to orchestrate.

## The non-negotiables (universal)

Any error-handling code is incomplete unless each of these is verified:

1. **Cause chain preserved.** Every re-wrap carries the original cause. `raise X from y`, `%w`, `Error.cause`, `Throwable.initCause`. No information-destroying re-wraps.
2. **No silent swallow.** Bare `except:`, `_ = result`, `if err != nil { return nil }`, `result.unwrap_or_default()` on a legitimately fallible operation — all flag as **must-fix** unless there's a written justification.
3. **Boundary classification.** At every propagation point, the error is classified as recoverable or terminal. Library code propagates; binary entry points decide.
4. **Cleanup on the error path.** RAII / `defer` / `try-finally` / `with` / `using` — verify resources are released when the failure happens, not only on the happy path.
5. **No library-level abort.** `panic!()`, `os.Exit()`, `process.exit()`, `System.exit()` — these belong only in `main` / binary entry points. Library code never aborts the host process. Exception: explicitly documented unrecoverable invariants (poisoned mutex, etc.).
6. **No log-and-rethrow.** Log once at the recovery boundary, not at every propagation step. Otherwise the same error is logged 5 times and operators can't tell what actually happened.
7. **No info leakage.** Stack traces, internal paths, secrets, PII in user-visible error messages — must-fix.
8. **For CLI output specifically:** defer to **`cdc-err`** for envelope/format. This skill stops at "an error reaches the binary boundary"; `cdc-err` takes over from there.

## Severity taxonomy

Standard five-class severity taxonomy used across most code-review workflows. See `references/severity.md` for the error-handling-specific criteria per band.

| Class | Error-handling example | Blocking? |
|---|---|---|
| **must-fix** | Silent swallow, panic in library, lost cause chain, leaked PII, missing cleanup on error path | Yes |
| **should-fix** | Inconsistent error type, missing context wrap, `except Exception` catch-all, panic in CLI `main` without recovery, log-and-rethrow | No, but recommended |
| **nit** | Sentinel vs typed where either works, error-message string style, log level choice | No |
| **question** | Ambiguous intent ("did you mean to drop this error?") | Info |
| **praise** | Well-structured cause chain, good typed-error hierarchy, correct resource cleanup, principled boundary classification | At least one per review |

## Decision rules in detail

### Review mode

1. Read the code. Note the language and inferred context (CLI binary / library / service / web handler).
2. Walk `references/checklist.md` in order. For each section, scan for the patterns it names.
3. For each hit, classify with `references/severity.md` and emit a finding using this format:

```
[must-fix] <Brief title>
  file: src/io/reader.rs:42
  what: Two `unwrap_or(Vec::new())` calls discard the underlying io::Error.
  why:  Callers cannot distinguish a successful empty read from a failed
        read that was silently turned into empty. Section 2 (Swallowing
        Detection) of the checklist.
  fix:  Change return type to Result<Vec<u8>, io::Error>; propagate with ?
        at the call sites in src/api.rs:88, src/api.rs:102.
```

4. At the end, include at least one **praise** finding if any error handling is done well — this is not optional; reviews that only flag negatives skew toward false confidence in the negatives.

### Refactor mode

Same as Review, but each **must-fix** and **should-fix** carries a concrete before/after diff. Diffs should be small and ready for the `Edit` tool to apply. Don't bundle multiple unrelated fixes into one diff.

### Design mode

The caller has no code yet — they're asking how to structure error handling for a new component. Load the relevant `references/languages/<lang>.md`, summarize the idiomatic pattern, name the standard library / popular crate / package recommendation, and flag the 2-3 most common mistakes for that language. If the new component is a CLI, hand off to `cdc-err` for envelope work.

### Work-Plan mode

Threshold: >5 findings OR findings span >3 files. Emit the YAML per `references/workplan.md`. Group findings into milestones if there's a natural clustering (e.g. "Error Hardening: Library Boundary", "Error Hardening: CLI Surface"). Set priority from severity (must-fix → P0, should-fix → P1, nit → P2). Estimate effort per finding (S / M / L / XL).

If the user then says "create the issues," **do not create them directly** — the YAML is the deliverable. Tell the user to feed it to whatever issue-creation flow their project uses, or create issues manually using the schema in `references/workplan.md`. Issue creation is not in this skill's scope.

## Output format examples

### Review-mode example

```
ERROR-HANDLING REVIEW — 4 findings (1 must-fix, 2 should-fix, 1 praise)

[must-fix] Swallowed I/O error in src/io/reader.rs:42
  what:  unwrap_or(Vec::new()) discards io::Error from read_to_end()
  why:   Section 2 (Swallowing Detection) — partial-read bugs invisible to callers
  fix:   Change signature to fn read_all(...) -> io::Result<Vec<u8>>, propagate with ?

[should-fix] Generic except Exception in worker.py:117
  what:  Catches ALL exceptions including KeyboardInterrupt and SystemExit
  why:   Section 5 (Library Purity) — masks user-initiated cancellation
  fix:   Replace with except (ApiError, TimeoutError); re-raise on Exception

[should-fix] log.Error + return err in handler.go:204
  what:  Logs the error then propagates it; caller logs it again at line 312
  why:   Section 8 (Logging Discipline) — log-and-rethrow produces duplicate log entries
  fix:   Remove the log here; the boundary in main.go owns logging

[praise] Cause chain preserved in api/errors.py:30-55
  what:  ApiError(__cause__=original) preserves the chain across all 3 wrap sites
  why:   Section 1 (Cause-Chain Preservation) — exactly the pattern PEP 3134 prescribes
```

### Refactor-mode diff example

```diff
- pub fn read_all(path: &Path) -> Vec<u8> {
-     std::fs::read(path).unwrap_or(Vec::new())
- }
+ pub fn read_all(path: &Path) -> io::Result<Vec<u8>> {
+     std::fs::read(path)
+ }
```

### Work-Plan-mode YAML — see `references/workplan.md` for the full schema.

## Out of scope for this skill

Tell the caller explicitly when they're asking outside the skill's territory:

- **Formatting, naming, performance, generic style.** Not this skill — use a general code-review tool for that.
- **Test coverage of error paths.** Adjacent — use a test-generation tool for that.
- **Security beyond error-related.** Information leakage in error messages **is** in scope. SQL injection, XSS, auth — not this skill.
- **Business-logic correctness.** Whether your `if` branches are right is not error handling.
- **CLI envelope shape, exit codes, dual rendering.** **Defer to `cdc-err`.**

## References

- `references/checklist.md` — 8-section audit checklist with detection patterns and fix templates.
- `references/severity.md` — taxonomy + error-handling-specific criteria per band.
- `references/workplan.md` — YAML schema for Work-Plan mode and the recommended issue-body shape for whatever issue-creation flow the user uses.
- `references/languages/{rust,go,python,typescript,java}.md` — load only the one that matches the code under review.

## Sibling skill

`cdc-err` covers CLI error *output*: RFC 9457 envelope, three mandatory agent extensions (`retry_after`, `suggested_fix`, `code_actions[]`), applicability markers, dual rendering (`--format` / TTY). When a `cdc-review` finding crosses into "how should this error appear to the human / agent at the CLI boundary", hand the work over.

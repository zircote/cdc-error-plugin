# Error-Handling Review Checklist

Walk these sections in order. Each section: **rule**, **why it matters**, **detection patterns**, **fix template**.

Findings format (used by the SKILL.md output rules):

```
[severity] <Brief title>
  file: <path:line>
  what: <one-line factual description>
  why:  <citation to this checklist section + the underlying principle>
  fix:  <one-line concrete remediation>
```

---

## 1. Cause-chain preservation

**Rule.** Every re-wrap of an error preserves the original cause.

**Why.** Operators reading a stack trace need to see the underlying I/O / parse / network error. Re-wraps that drop the cause turn diagnostic minutes into diagnostic hours and hide the actual failure mode.

**Detection patterns (by language):**

- **Python:** `raise SomeError(...)` inside an `except` block without `from` (or `from None` only if explicit suppression is documented). PEP 3134.
- **Rust:** `Result<..., MyError>` constructed by `.map_err(|_| MyError::Generic)` — `_` discards the upstream error. Acceptable only when `MyError` carries the cause via a `#[from]` source field or a `Box<dyn Error>`.
- **Go:** `fmt.Errorf("...: %s", err)` — string-formats the inner error and loses `errors.Is` / `errors.As` capability. Should be `%w`.
- **TypeScript/Node:** `throw new ApiError("...")` inside a `catch (e)` block without `{ cause: e }` (ES2022).
- **Java:** `throw new MyException("...")` from a `catch (Throwable t)` without `initCause(t)` or constructor-passing.

**Fix template.**
- Python: `raise NewError("context") from e`
- Rust: `#[from]` on the source variant, or `MyError::from_source(e)`
- Go: `fmt.Errorf("context: %w", err)`
- TS: `throw new ApiError("context", { cause: e })`
- Java: `throw new MyException("context", t)`

**Severity default:** must-fix when the wrap destroys diagnostic information; should-fix when an alternative diagnostic channel exists (e.g. structured logging already captures the cause).

---

## 2. Swallowing detection

**Rule.** No error is silently ignored.

**Why.** A swallowed error turns a recoverable failure into a silent corruption. The system continues running with bad state.

**Detection patterns:**

- **Python:** `except:` (bare) or `except Exception: pass`. Also `try: ... except: log.debug(...)` with no re-raise and no recovery path.
- **Rust:** `let _ = fallible()`, `.unwrap_or_default()` on operations that can legitimately fail, `.ok()` followed by `.unwrap()` on `Option`, `if let Ok(_) = ...` with no `else`.
- **Go:** `if err != nil { return nil }`, `_ = fallible()`, `if err != nil { /* TODO */ }`.
- **TypeScript:** `try { ... } catch {}`, `try { ... } catch (_) {}`, `.catch(() => {})` on a Promise.
- **Java:** empty `catch (Exception e) {}`, `catch (Exception e) { /* ignored */ }`.

**Exceptions to the rule (acceptable swallows):**

- A documented cleanup path where the cleanup error is genuinely less important than the original. Must have a comment naming the principal cause.
- Polling loops where one tick failing is expected (must log at debug+, never silent).
- Code paths after a panic/abort decision where the swallow is intentional.

**Fix template.** Propagate to a real boundary, or replace with an explicit `recover-and-log-with-cause-chain` and document why.

**Severity default:** must-fix.

---

## 3. Boundary classification

**Rule.** Every propagation point carries a decision: is this error recoverable, terminal, or a programmer bug?

**Why.** Without classification, the binary entry point can't make a correct retry/abort decision, and library callers can't distinguish "expected, retry" from "broken, escalate."

**Detection patterns:**

- A `Result<T, Box<dyn Error>>` / `Result<T, anyhow::Error>` / `interface{} error` / `RuntimeException` everywhere — these flatten the classification.
- One enormous `MyError` enum with 30+ variants — flag for splitting into recoverable / programmer-bug categories.
- Library code returning `Option<T>` for fallible operations where the caller can't distinguish "not found" from "failed to look up."

**Fix template.** Introduce typed error categories at API boundaries. Rust: an `enum` with explicit variants. Go: sentinel errors with `errors.Is` checking. Python: a small custom hierarchy. TypeScript: typed `Error` subclasses with `name` discriminators. Java: separate checked vs unchecked.

**Severity default:** should-fix.

---

## 4. Resource cleanup on the error path

**Rule.** Every resource acquired on the happy path is released on every error path.

**Why.** Connection leaks, file handle exhaustion, locks held forever — these are the bugs that show up at 3am, not in your test suite.

**Detection patterns:**

- **Python:** A `try` that opens/locks/connects without a matching `finally` or `with`. `with` is strongly preferred.
- **Rust:** Largely handled by RAII / `Drop`. Flag manual `unsafe { free(...) }` paths; flag `mem::forget` near `Drop` implementations.
- **Go:** Allocation without an immediate `defer cleanup()`. Look for `os.Open` / `sql.DB.Begin` / `net.Dial` without a `defer` on the next line.
- **TypeScript:** `await connect()` without a `try { ... } finally { await close() }` block. Prefer `using` (TC39 stage 3 / TS 5.2+) when available.
- **Java:** Resource acquisition outside a `try-with-resources` block.

**Fix template.** Convert to the language's RAII / scoped-cleanup idiom: `with`, `defer`, `try-with-resources`, `using`, Rust `Drop`.

**Severity default:** must-fix for resources with system-wide impact (DB connections, file handles, locks). should-fix for in-process resources.

---

## 5. Library purity (no process abort)

**Rule.** Library code never calls `panic!()` / `os.Exit()` / `process.exit()` / `System.exit()`. Only binary entry points decide to abort.

**Why.** Library aborts make a library unusable in any host that needs to handle the error: tests, async runtimes, supervisors, REPLs. A library that aborts is a library that can't be composed.

**Detection patterns:**

- **Rust:** `panic!()`, `unwrap()`, `expect()` in `src/lib.rs` or any module reachable from it. Acceptable in `src/main.rs` and in `#[test]` code. Also flag `.unwrap()` on `Result` types in trait methods declared on public types.
- **Go:** `panic(...)`, `log.Fatal(...)`, `os.Exit(...)` outside `main.go`.
- **Python:** `sys.exit(...)`, `os._exit(...)` outside the entry-point module.
- **TypeScript/Node:** `process.exit(...)` outside the CLI entry script.
- **Java:** `System.exit(...)` outside `main`. `throw new Error(...)` where `Error` is the Java unchecked top type.

**Exceptions:** documented unrecoverable invariants (poisoned mutex, broken global state) where continuing is unsafe. Must have a comment naming the invariant.

**Fix template.** Return a `Result` / `error` / typed exception. Let the caller decide.

**Severity default:** must-fix.

---

## 6. Information leakage in error messages

**Rule.** User-visible error messages do not contain secrets, internal paths, raw SQL, PII, or stack traces.

**Why.** Error messages reach logs, tickets, screenshots, agent transcripts. A connection string with an embedded password in an error message is a credential leak.

**Detection patterns:**

- String-formatting an `os.environ` / `process.env` value into an error message.
- Surfacing a raw SQL query that includes user-supplied parameters into the error.
- Printing a full stack trace to `stderr` in a production binary without a "debug mode" gate.
- Including DB row contents in a "constraint violation" error.

**Fix template.** Split into two surfaces: a user-visible message (generic, with a correlation ID) and an internal log (full detail, only reachable by operators).

**Severity default:** must-fix. Treat any candidate leak as must-fix until proven safe.

---

## 7. CLI output shape (deferred to cdc-err)

**Rule.** Errors that reach a CLI binary boundary follow the dual-consumer pattern: RFC 9457 envelope, three mandatory agent extensions, applicability markers, dual rendering.

**Why.** That's the territory of the sibling skill. This checklist's job is to confirm the error *arrives* at the boundary in good shape; `cdc-err` decides what the boundary emits.

**Detection:** If the code under review is in a CLI's `main()` or `cmd/`/`bin/` directory and constructs error output for the user/agent, that's `cdc-err` territory — name the finding and hand off.

**Severity:** N/A — defer.

---

## 8. Logging discipline

**Rule.** Each error is logged exactly once, at the recovery boundary (the point where the error is handled, not where it's propagated).

**Why.** Log-and-rethrow produces duplicate log lines for the same incident. Operators can't tell whether they're looking at one bug or many.

**Detection patterns:**

- `log.error(...); raise` / `log.error(...); return err` / `console.error(...); throw` patterns at every level of the call stack.
- A logged error that doesn't include the cause chain.
- Logging at the wrong level (e.g. `error` for an expected-failure 404).

**Fix template.**
- In propagation positions: just propagate. No log.
- At the boundary (request handler, CLI main, top-level job loop): log once with the full cause chain.

**Severity default:** should-fix.

---

## After the walk

If the checklist produces:
- 0 must-fix and 0 should-fix → state that explicitly; still emit at least one **praise**.
- 1-5 findings → emit Review-mode report directly.
- 6+ findings or findings span >3 files → switch to **Work-Plan mode** (see `workplan.md`).
- The review touches CLI output → cross-link to `cdc-err` for those findings.

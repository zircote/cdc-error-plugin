# Rust Error Handling — Review Reference

Load this file when the code under review is Rust. Cites the Rust API guidelines and the `thiserror` / `anyhow` ecosystem conventions per principle.

## Idiomatic patterns to accept

- **`Result<T, E>` returned from any fallible function.** No exceptions, no out-params.
- **Custom error type per crate-public API** built with `thiserror::Error` + `#[derive(Debug)]`. Variants name the failure mode (`#[error("could not parse config: {0}")]`), not the implementation detail.
- **`#[from]` on source variants** so the cause chain travels through `std::error::Error::source()`.
- **`?` for propagation.** Multiple `?` chains are fine.
- **`anyhow::Result` in application/binary code** for ergonomic error context (`anyhow::Context::with_context`). Acceptable at the binary entry point and in test code — **not** in library public APIs.
- **`Result<T, ()>`** only when the error genuinely carries no information (rare). Prefer a typed variant.
- **`unwrap()` / `expect()` in `main()` after dual-consumer envelope emission**, or in `#[test]` code, or after a documented invariant check (`assert!`).

## Anti-patterns (must-fix / should-fix)

### must-fix

- **`panic!()` / `unwrap()` / `expect()` in library code** reachable from `src/lib.rs`. Section 5 of `checklist.md`.
- **`.map_err(|_| MyError::Generic)`** — discards the upstream error. Use `#[from]` or `MyError::from_source(e)` to preserve the chain.
- **`let _ = fallible();`** outside a documented swallow context. Section 2.
- **`Box<dyn Error>` at a public API boundary** for a library where callers need to discriminate failure modes. Section 3.
- **`.lock().unwrap()` in library code.** `PoisonError` is a recoverable signal, not an abort condition.

### should-fix

- **`anyhow::Error` in a library's public API.** Use a typed error so callers can branch on variants.
- **`Result<T, String>` or `Result<T, &'static str>`.** Strings can't be matched on, can't carry context.
- **`.unwrap_or_default()` on `Result<T, E>`** where `E` could be a real failure mode. The default value silently masks the error.
- **Re-wrapping with `format!("{}", e)`** — drops the underlying error type. Use `#[source]` field.

### nit

- Order of variants in a `#[derive(thiserror::Error)]` enum.
- Choice between `anyhow::Context` and `with_context()` (closures are slightly more efficient when the context is lazy).
- Whether to use `impl Error` vs the type by name in trait bounds.

## Cause-chain inspection

Reviewers checking whether the cause chain is preserved should mentally walk:

```rust
let mut current: Option<&dyn Error> = Some(&top_err);
while let Some(e) = current {
    println!("{}", e);
    current = e.source();
}
```

If `source()` returns `None` immediately on the wrapped error, the chain is broken.

## Boundary patterns

- **CLI `main()`** — pattern-match the top-level error to decide exit code and envelope. **Defer to `cdc-err`** for the envelope itself.
- **Library entry point** — return `Result<T, MyError>` with typed variants. Callers match.
- **Async (`tokio`, `async-std`)** — same rules. `JoinError` is not a swallow — it represents task panic and must be surfaced.

## Detection commands (Rust-specific grep patterns)

```bash
# Library panic/unwrap candidates (filter out tests + main)
rg '\.unwrap\(\)|\.expect\(' src/ --type rust \
  | rg -v 'src/main\.rs|src/bin/|#\[test\]|#\[cfg\(test\)\]'

# String-formatted error losses
rg 'map_err\(\|_\|' src/ --type rust
rg 'format!\("\{[^}]*\}: \{e\}"' src/ --type rust   # likely lost cause

# Library aborts
rg 'panic!\(|process::exit|std::process::exit' src/ --type rust \
  | rg -v 'src/main\.rs|src/bin/'
```

## Recommended crates

- **`thiserror`** for library error types.
- **`anyhow`** for binary/application error handling.
- **`miette`** when the error needs both human and JSON rendering — see sibling skill `cdc-err`.

## Cross-link to `cdc-err`

For CLI `main()` error output (dual rendering, RFC 9457 envelope, `--format=json`), see `cdc-err/references/languages.md` Rust section. This skill stops at "the error arrives at `main()` in good shape"; `cdc-err` takes over for what `main()` emits.

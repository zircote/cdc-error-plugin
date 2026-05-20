# Go Error Handling — Review Reference

Load this file when the code under review is Go. Cites the `errors` package guidance and the 2019 error-values design notes.

## Idiomatic patterns to accept

- **Errors as values.** Every fallible function returns `(T, error)` (or `(T1, T2, error)` for multi-result).
- **Sentinel errors** (`var ErrNotFound = errors.New("not found")`) for stable, matchable conditions.
- **Typed errors** (`type ParseError struct { ... }`) when callers need data from the error.
- **`%w` for wrapping** so `errors.Is` and `errors.As` still work up the chain.
- **`errors.Is(err, target)`** for sentinel comparison.
- **`errors.As(err, &target)`** for typed extraction.
- **`errors.Join(errs...)`** (Go 1.20+) when reporting multiple failures from a batch operation.
- **`defer cleanup()`** immediately after every resource acquisition.

## Anti-patterns

### must-fix

- **`if err != nil { return nil }`** — swallow. Section 2.
- **`_ = fallible()`** when the error matters. Acceptable for `_ = file.Close()` in cleanup paths *if* documented.
- **`log.Fatal(...)` / `os.Exit(...)` / `panic(...)` outside `main()`.** Section 5.
- **`fmt.Errorf("...: %s", err)`** — loses `errors.Is` chain. Must be `%w`.
- **Returning `nil` after a non-nil error** — caller can't tell which to check. Section 3.

### should-fix

- **`if err != nil { return err }`** without wrapping at API boundaries. The caller gets no operational context. Wrap with `fmt.Errorf("doing X: %w", err)`.
- **Returning `errors.New("descriptive text")` inline at multiple call sites.** Promote to a package-level sentinel or typed error.
- **Logging the error AND returning it.** Log-and-rethrow. Section 8.
- **`type Error interface { ... }` overriding the standard `error`** — generally avoid; use `errors.As` instead.
- **`recover()` outside a top-level boundary** — swallows panics that should be observed.

### nit

- Trailing capitalization / period in error message strings. Go convention: lowercase, no period (`fmt.Errorf("read failed: %w", err)` not `"Read failed."`).
- `errors.Is` order when checking multiple sentinels — only matters if mutually exclusive.
- Sentinel vs typed when both work.

## Patterns by layer

- **Handlers / RPC layer:** wrap with operation context (`"handling /api/users: %w"`), return to framework which formats. Log once at framework boundary.
- **Service layer:** wrap with business context; don't log.
- **Repository / I/O layer:** return raw library error (`io.ErrUnexpectedEOF`, `pgx.ErrNoRows`); let upper layers decide classification.
- **`main()`:** the only place `log.Fatal` is acceptable, and even then prefer `os.Exit(code)` after emitting a proper error envelope. See `cdc-err`.

## Detection commands

```bash
# Swallows
rg 'if err != nil \{\s*return nil' --type go
rg '_ = .*\bfallible\b' --type go     # naming heuristic; adapt to repo

# Lost cause chain
rg 'fmt\.Errorf\([^)]*%s[^)]*, err\)' --type go
rg 'errors\.New\([^)]+\bfailed\b[^)]+%v' --type go

# Library aborts
rg 'log\.Fatal|os\.Exit|panic\(' --type go \
  | rg -v '_test\.go|/main\.go|cmd/.*\.go'

# Missing wrap
rg 'return err$' --type go | head -20    # spot-check; sometimes correct, sometimes lazy
```

## Recommended packages

- **stdlib `errors`** — sufficient for most cases since Go 1.13.
- **`github.com/pkg/errors`** — deprecated; migrate to stdlib `fmt.Errorf` with `%w`.
- **`github.com/cockroachdb/errors`** — richer cause chains, structured fields. Optional but well-supported.

## Cross-link to `cdc-err`

CLI binary error output (Cobra, exit codes, `--format=json`, RFC 9457 envelope) belongs to `cdc-err`. `cdc-review` covers what happens **before** `main()` decides how to render — the propagation, wrapping, and cleanup choices.

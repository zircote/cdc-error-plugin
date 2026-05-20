# Python Error Handling — Review Reference

Load this file when the code under review is Python. Cites PEP 3134 (exception chaining), the `contextlib` docs, and the cpython exception hierarchy.

## Idiomatic patterns to accept

- **Specific `except` clauses** (`except (IOError, OSError)`) — narrow as the failure mode being handled.
- **`raise NewError("context") from e`** at every re-wrap. PEP 3134.
- **Custom exception hierarchy** rooted at one project-level base (`class MyAppError(Exception)`).
- **`with` statement** for every resource acquisition.
- **`contextlib.suppress(SpecificError)`** for documented intentional swallows.
- **`raise ... from None`** when the chain should be deliberately suppressed (rare, must be documented).
- **Logging at the recovery boundary only**, using `logger.exception()` to capture the chain.

## Anti-patterns

### must-fix

- **Bare `except:` or `except: pass`.** Catches `KeyboardInterrupt`, `SystemExit`, `MemoryError`. Section 2.
- **`raise NewError(...)` inside `except` without `from`** — drops the cause. Section 1.
- **`sys.exit(...)` / `os._exit(...)` outside the entry-point module.** Section 5.
- **`try: ... except Exception: log.error(...)` with no re-raise** and no recovery — silent corruption.
- **Catching `Exception` and continuing** at a layer that can't actually recover.

### should-fix

- **`except Exception:`** at non-boundary layers. Too broad — catches `TypeError`, `KeyError`, framework internals. Narrow to the specific exception type.
- **Custom exceptions inheriting from `BaseException`** — should inherit from `Exception`.
- **`raise ValueError("...")` re-wrapping an existing exception** without `from e`.
- **Per-line resource opening without `with`** (`f = open(...)` outside a context manager).
- **`logger.error(msg); raise`** — log-and-rethrow. Section 8.

### nit

- `logger.error` vs `logger.exception` (the latter auto-captures the traceback — usually preferred).
- Order of `except` blocks when they don't overlap (most-specific-first is conventional but optional).
- Whether to include a custom `__str__` on exception subclasses.

## Boundary layers

- **Framework handlers (FastAPI, Flask, Django):** let the framework's exception middleware translate to HTTP. Add typed exceptions for known error classes (`class NotFoundError(MyAppError)`).
- **CLI entry point:** the only place `sys.exit(...)` should appear. For Typer/Click, prefer raising a typed exception and letting the framework's error handler call `sys.exit` after emitting an envelope. **Defer to `cdc-err`.**
- **Library code:** raise typed exceptions. Never `sys.exit`.
- **Async (`asyncio`, `trio`):** `asyncio.CancelledError` is a control signal, not an error — never swallow it, re-raise after cleanup.

## Cause chain inspection

The reviewer should walk:

```python
e = top_error
while e is not None:
    print(type(e).__name__, e)
    e = e.__cause__ or e.__context__
```

`__cause__` is set by `raise ... from e`. `__context__` is set implicitly when an exception is raised inside `except`. A chain that's all `__context__` and no `__cause__` is a signal that someone forgot the `from`.

## Detection patterns

```bash
# Bare/broad except
rg '^\s*except:\s*$|^\s*except\s*Exception\s*:' --type py

# Missing 'from' in re-raise (heuristic; manual confirm)
rg -A 3 '^\s*except' --type py | rg 'raise [A-Z]\w+\(' | rg -v 'from'

# Library aborts
rg 'sys\.exit\(|os\._exit\(' --type py \
  | rg -v 'cli\.py|__main__\.py|test_'

# Resource without 'with'
rg '\bopen\(' --type py | rg -v 'with\s+open|@contextmanager'
```

## Recommended libraries

- **stdlib only** is usually sufficient.
- **`structlog`** for structured logging at the boundary.
- **`tenacity`** for retry logic where the error is classified as recoverable.

## Cross-link to `cdc-err`

CLI envelope for Click/Typer (Rich human renderer + `application/problem+json` JSON renderer + `--format` selection + TTY detection) is `cdc-err` territory. This skill covers the exception-propagation choices that happen **before** the CLI boundary.

# TypeScript / Node.js Error Handling — Review Reference

Load this file when the code under review is TypeScript or Node.js. Cites the ES2022 `Error.cause` proposal, the Node.js error conventions, and the `neverthrow` Result pattern.

## Idiomatic patterns to accept

- **`Error` subclasses with discriminator** (`class ApiError extends Error { name = "ApiError" }`).
- **`new Error("...", { cause: original })`** — ES2022 cause chaining.
- **`try/catch` with typed narrowing** (`if (err instanceof ApiError)`).
- **`async`/`await` with `try`/`catch`** instead of `.then().catch()`.
- **`neverthrow`'s `Result<T, E>`** for callers who want explicit error handling without exceptions (acceptable, not required).
- **`try/finally` or TC39 `using` (TS 5.2+)** for resource cleanup.
- **Logging at the boundary** (`process.on("uncaughtException")`, Express error middleware, etc.).

## Anti-patterns

### must-fix

- **`try { ... } catch {}` / `catch (_) {}`** — silent swallow. Section 2.
- **`.catch(() => {})`** on a Promise — same swallow, async flavor.
- **`process.exit(...)` outside CLI entry script.** Section 5.
- **`throw new Error("...")` inside `catch (e)` without `{ cause: e }`.** Drops chain.
- **Unhandled Promise rejections** that are *intentionally* unhandled (`promise.then(...)` with no `.catch` at the top level). Node's default is now to crash on unhandled rejection — relying on this is fragile.

### should-fix

- **`throw "string literal"`** — not an Error, has no stack trace.
- **`catch (e: any)`** — should be `catch (e: unknown)` (TS 4.4+ default) with explicit narrowing.
- **`Error` re-thrown as a different class without preserving `cause`.**
- **Mixing callback-style errors (`cb(err, result)`) with Promise rejection** in the same module.
- **`console.error(err); throw err`** — log-and-rethrow. Section 8.

### nit

- `Error` subclass naming convention (PascalCase + `Error` suffix is conventional but optional).
- Whether to set `name` and `prototype` manually in older TS targets.
- `catch (e: unknown)` vs `catch (e)` (the latter defaults to `unknown` in modern TS).

## Async pitfalls

- **`Promise.all`** rejects on the first failure, silently dropping the others. Use `Promise.allSettled` when you need all results.
- **Forgotten `await`** in an `async` function — the error becomes an unhandled rejection. Frequent must-fix.
- **`AbortController.signal.throwIfAborted()`** — the cancel path. Don't swallow `AbortError`.

## Type-narrowing patterns the reviewer should accept

```ts
try {
  await doThing();
} catch (e: unknown) {
  if (e instanceof ApiError) {
    // handle typed
  } else if (e instanceof Error) {
    throw new WrapperError("doing thing failed", { cause: e });
  } else {
    throw new WrapperError(`doing thing failed: ${String(e)}`);
  }
}
```

## Detection patterns

```bash
# Silent catches
rg 'catch \(_\) \{|catch \{\}|\.catch\(\(\) => \{\}\)' --type ts --type js

# String throws
rg 'throw\s+["'"'"'`]' --type ts --type js

# Library-level exit
rg 'process\.exit\(' --type ts --type js \
  | rg -v 'bin/|cli\.|__main__'

# Lost cause
rg -A 1 '^\s*catch' --type ts | rg 'throw new' | rg -v 'cause'
```

## Node-specific considerations

- **`EventEmitter`'s `"error"` event** — if no listener, the process crashes. Always attach a listener on long-lived emitters.
- **`stream`s** — pipe error propagation requires `pipeline()` from `stream/promises` (Node 15+), not raw `.pipe()`.
- **HTTP server errors** — `server.on("clientError", ...)` and `server.on("error", ...)` need explicit handlers.

## Recommended libraries

- **stdlib** is sufficient for most cases.
- **`neverthrow`** for explicit `Result<T, E>` style where preferred.
- **`pino` / `winston`** for structured boundary logging.
- **`zod` / `valibot`** — parse errors carry structured info; preserve their cause chain on wrap.

## Cross-link to `cdc-err`

For a Node CLI's error envelope (Commander/Yargs handler + dual rendering + RFC 9457 + `--format=json` flag + TTY detection), the post the sibling skill is built on is silent on Node's idiomatic library — `cdc-err` will recommend the dual-format pattern but won't name a specific library. Use whatever your CLI framework provides.

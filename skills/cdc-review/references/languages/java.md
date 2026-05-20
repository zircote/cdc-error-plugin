# Java Error Handling — Review Reference

Load this file when the code under review is Java (or Kotlin/Scala on the JVM). Cites *Effective Java* items 69-77 on exception design.

## Idiomatic patterns to accept

- **Checked exceptions for recoverable conditions** the caller is expected to handle.
- **Unchecked exceptions (`RuntimeException` subclasses) for programmer errors** — bugs, contract violations.
- **`Throwable.initCause(...)` or constructor `super(message, cause)`** preserving the cause chain.
- **`try-with-resources`** for every `AutoCloseable` resource.
- **Multi-catch (`catch (IOException | ParseException e)`)** when handling is the same.
- **Custom exception classes per business failure mode**, inheriting from a project base (`extends MyAppException`).

## Anti-patterns

### must-fix

- **`catch (Exception e) {}`** or `catch (Throwable t) {}` with empty body. Section 2.
- **`catch (Throwable t)` at non-boundary layer** — Throwable includes `OutOfMemoryError`, `StackOverflowError`, which should usually propagate.
- **`System.exit(...)` in library code.** Section 5.
- **`throw new MyException("...")` inside `catch (Throwable t)` without passing `t`** as the cause. Lost chain. Section 1.
- **Returning `null` after catching an exception** — caller can't distinguish absence from failure. Section 3.

### should-fix

- **`catch (Exception e) { throw new RuntimeException(e); }`** — converts checked to unchecked without intent. Often masks the real problem.
- **Empty `finally` blocks** — usually a sign that cleanup was intended but forgotten.
- **`printStackTrace()`** outside top-level boundary — should be a logger call.
- **`log.error(...) throw ...`** — log-and-rethrow. Section 8.
- **Catching `Exception` to convert to `Optional.empty()`** — same swallow-with-different-shape.

### nit

- Whether to use `Throwables.getRootCause(t)` from Guava vs walking `getCause()` manually.
- Checked vs unchecked for borderline cases (e.g. `JsonProcessingException`).
- Multi-catch vs separate catches when the bodies differ only slightly.

## Resource cleanup — `try-with-resources` is mandatory

```java
// Required pattern:
try (FileInputStream in = new FileInputStream(path)) {
    // ...
} catch (IOException e) {
    throw new ConfigLoadException("loading " + path, e);
}

// Anti-pattern:
FileInputStream in = null;
try {
    in = new FileInputStream(path);
    // ...
} finally {
    if (in != null) in.close();   // can itself throw, masking the original
}
```

The reviewer should flag every `finally` block with a `.close()` call that could itself throw — that's a must-fix unless the close error is explicitly suppressed.

## Exception design

- **Don't make a one-off `RuntimeException` per failure mode.** Group into a small hierarchy with discriminator.
- **`getCause()` should always return the real cause** if there was one upstream.
- **Don't override `getMessage()` to do something expensive** — it's called by logging frameworks repeatedly.
- **`Throwable.getSuppressed()`** matters for try-with-resources — closure exceptions are suppressed, not lost.

## Boundary patterns

- **Spring/Quarkus controllers:** let the framework's `@ControllerAdvice` translate exceptions to HTTP responses. Add typed exceptions for known classes.
- **CLI `main()`:** the only place `System.exit(code)` is acceptable. Catch the top-level exception, log once with full chain, exit with a meaningful code. **Defer envelope shape to `cdc-err`.**
- **Library code:** declare checked exceptions in the throws clause; let callers handle.
- **Async (`CompletableFuture`):** `exceptionally` / `handle` are the recovery points. Don't `.join()` and swallow `CompletionException`.

## Detection patterns

```bash
# Silent swallows
rg -A 1 '^\s*catch \(' --type java | rg '^\s*\}\s*$'

# System.exit in library code
rg 'System\.exit\(' --type java \
  | rg -v 'Main\.java|Cli\.java|src/test/'

# Lost cause: throw inside catch without 'e' passed
rg -A 2 'catch \([A-Z]\w+ e\)' --type java | rg 'throw new' | rg -v '\(.*e\)'

# Resource not in try-with-resources
rg 'new FileInputStream|new FileOutputStream|Connection.*=' --type java \
  | rg -v 'try\s*\('
```

## Recommended libraries

- **stdlib `java.util.logging` or SLF4J facade** for boundary logging.
- **Guava `Throwables`** for chain walking utilities.
- **Resilience4j** for retry/circuit-breaker patterns where errors are classified as recoverable.

## Kotlin specifics

- **`Result<T>`** (`kotlin.Result`) is a useful local-only return type — should not cross module boundaries (per stdlib design).
- **`runCatching { ... }.getOrElse { ... }`** — acceptable when the recovery is explicit. `.getOrNull()` swallows.
- **`Throwable.printStackTrace()`** in Kotlin code outside `main` — same anti-pattern as Java.

## Cross-link to `cdc-err`

JVM CLIs (Picocli, Spring Shell, etc.) need the dual-rendering pattern at `main()`. `cdc-err` doesn't name a specific JVM library — the post is silent on Java/Kotlin — but the envelope contract is the same. This skill stops at "the exception reaches `main()`"; `cdc-err` takes over for what `main()` emits.

# Work-Plan Mode

Used when a review produces >5 findings OR findings span >3 files. The skill emits a YAML decomposition the user can review and then hand off to whatever issue-creation flow their project uses.

This skill does **not** create issues directly. It produces the YAML; the user or a tool of their choice takes it from there. The schema below is designed to map cleanly to GitHub issues but works for any tracker that accepts free-form titles + bodies.

## YAML schema

```yaml
project: <plugin or repo name>
generated_by: cdc-review
review_scope: <one-line summary of what was reviewed>

milestones:
  - name: "Error Handling Hardening: Library Boundary"
    rationale: "Library code currently panics on poisoned mutex and swallows I/O errors. Must be fixed before downstream consumers can rely on the library."
  - name: "Error Handling Hardening: CLI Surface"
    rationale: "CLI main emits raw tracebacks. Defer envelope work to cdc-err."

findings:
  - id: EH-001
    title: "Swallowed I/O errors in src/io/reader.rs"
    severity: must-fix
    files:
      - src/io/reader.rs:42
      - src/io/reader.rs:89
    problem: |
      Two `unwrap_or(Vec::new())` calls discard the underlying io::Error
      from `std::fs::read`, making partial-read bugs invisible to callers.
    expected: |
      Return `io::Result<Vec<u8>>`, propagate to the API boundary in
      src/api.rs where retry/abort can be decided.
    acceptance:
      - All call sites of `read_all` updated to handle the Result.
      - New test covers the partial-read failure mode.
      - No `unwrap_or` on fallible reads in src/io/.
    effort: M           # S = <1h, M = 1-4h, L = 4-16h, XL = >16h
    dependencies: []
    labels:
      - error-handling
      - must-fix
      - refactor
    milestone: "Error Handling Hardening: Library Boundary"
    priority: P0        # must-fix → P0, should-fix → P1, nit → P2

  - id: EH-002
    title: "Library panic on poisoned mutex in src/cache.rs:204"
    severity: must-fix
    files:
      - src/cache.rs:204
    problem: |
      `.lock().unwrap()` panics if any holder panicked. Library code must
      not abort the host process.
    expected: |
      Return a custom error variant on PoisonError; caller decides whether
      to abort or recover by reinitializing the cache.
    acceptance:
      - Custom error variant added with `#[from] PoisonError` source.
      - All `.lock().unwrap()` in library code replaced.
      - Tests verify recovery path.
    effort: L
    dependencies: []
    labels:
      - error-handling
      - must-fix
      - library-purity
    milestone: "Error Handling Hardening: Library Boundary"
    priority: P0

  - id: EH-003
    title: "log-and-rethrow in handler.go:204 and 312"
    severity: should-fix
    files:
      - internal/handler/handler.go:204
      - internal/handler/handler.go:312
    problem: |
      Errors are logged at the call site AND re-logged when they reach main.
      Operators see two log lines per incident, hard to correlate.
    expected: |
      Remove log calls at propagation sites; log once at main's recovery
      boundary with full cause chain.
    acceptance:
      - log.Error / slog.Error calls removed from middleware-style functions.
      - main wraps the top call with a single error log including `%+v` cause.
    effort: S
    dependencies: [EH-001]    # depends on EH-001 if the error type needs to expose the chain
    labels:
      - error-handling
      - should-fix
    milestone: "Error Handling Hardening: Library Boundary"
    priority: P1
```

## Field semantics

- **`id`** — `EH-NNN` sequential per review. Carry through to issue titles for traceability.
- **`severity`** — must-fix / should-fix / nit. Drives `priority` automatically (P0 / P1 / P2).
- **`files`** — `path:line` form. Multiple lines OK when the finding is the same pattern at multiple sites.
- **`problem` / `expected` / `acceptance`** — three required blocks. `problem` quotes/describes the current code, `expected` describes the target state, `acceptance` is a checklist a PR can be measured against.
- **`effort`** — S/M/L/XL using the wall-clock band shown above. Calibrated for one engineer with context.
- **`dependencies`** — other finding IDs that must land first. Form a DAG; reject cycles.
- **`labels`** — start with `error-handling` and the severity; add language/area tags as relevant.
- **`milestone`** — group related findings. Don't create a milestone per finding; aim for 2-4 milestones per review.
- **`priority`** — P0 must-fix, P1 should-fix, P2 nit. Allow promotion if the user signals urgency.

## Handing off the YAML

After producing the YAML, tell the user:

> Work plan ready. The YAML above is the deliverable. To turn each finding into a tracker issue, feed the YAML to whatever issue-creation flow your project uses, or create issues manually using the body shape below. This skill does not create issues directly — issue creation is out of scope.

Do not auto-invoke any external tool. The user reviews the YAML and decides how to act on it.

## Recommended issue body shape

If the user (or their tooling) wants a default issue body per finding, this shape maps cleanly to GitHub-style trackers and works for most others:

```
## Problem
<problem block>

## Expected behavior
<expected block>

## Affected files
- path:line
- path:line

## Acceptance criteria
- [ ] item
- [ ] item

## Dependencies
Blocked by #N (where #N is the issue created from finding EH-NNN)

## Source review
Generated by cdc-review on <date>. Finding ID: EH-NNN.
```

The mapping from YAML fields to the issue body:

| YAML field | Issue location |
|---|---|
| `title` | issue title |
| `problem` | "## Problem" section |
| `expected` | "## Expected behavior" section |
| `files` | "## Affected files" list |
| `acceptance` | "## Acceptance criteria" checklist |
| `dependencies` | "## Dependencies" — translate `EH-NNN` IDs to the created issue numbers |
| `labels` | tracker labels |
| `milestone` | tracker milestone |
| `priority` | tracker priority field, or a `priority:P0` label if priority is not a first-class field |

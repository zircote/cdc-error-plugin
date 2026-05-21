---
diataxis_type: explanation
title: Why three sibling skills, not one
---

# Why three sibling skills, not one

Error handling around a CLI tool has three surfaces:

1. **What goes out the door** — the bytes on `stderr`, the exit code, the JSON envelope an agent sees.
2. **What happens upstream of the door** — exception propagation, cause-chain preservation, panic discipline, resource cleanup.
3. **What happens on the other side of the door** — how the LLM agent receiving the envelope decides whether to retry, apply a fix, or hand back to a human.

A single skill that tried to do all three would either be too vague to give crisp advice, or so large it would dilute its triggers. So the plugin ships three siblings with disjoint scopes and explicit handoffs.

## The split

| Skill | Owns | Doesn't own |
|---|---|---|
| `cdc-err` *(producer)* | The envelope, the renderer, the selection logic, the exit-code mapping | Whether `try/except Exception: pass` upstream is OK; what the agent does with the envelope |
| `cdc-review` *(propagation)* | Propagation, swallowing, cause chains, panics, resource cleanup, language idioms | What the JSON envelope should look like; what the agent does with it |
| `cdc-handle` *(consumer)* | Parsing the envelope; choosing retry / apply_fix / handback / abort; gating actions on applicability markers; emitting an executable action plan | What the envelope should contain; whether the code that produced it has good propagation |

Each skill names the others when the work crosses the boundary:

- `cdc-review` defers envelope/format questions to `cdc-err` and consumer-side action questions to `cdc-handle`.
- `cdc-err` defers code-propagation questions to `cdc-review` and consumer-side decisions to `cdc-handle`.
- `cdc-handle` defers producer-side schema questions to `cdc-err` and source-code propagation questions to `cdc-review`.

This is not duplication. It is three angles on the same incident: the producer (`cdc-err`) writes the contract, the code review (`cdc-review`) keeps the upstream from violating it, and the consumer (`cdc-handle`) honors the contract on the receiving end. The contract is what makes the three skills compose — each is loyal to the same envelope shape.

## Why the boundary is at "the door"

The dual-consumer framing (see [Why CLI errors are a dual-consumer problem](dual-consumer.md)) is CLI-specific. Once you cross the door inward, the same error-handling principles apply whether the surface is a CLI, a library, a service, or a long-running daemon. `cdc-review` is language-agnostic and context-agnostic for that reason; `cdc-err` is intentionally CLI-only.

If you ask `cdc-err` about a REST API's error format, it declines and points at RFC 9457 directly. If you ask `cdc-review` to design a new error envelope, it declines and names `cdc-err`.

## When both fire

Requests like *"review my Cobra command — propagation **and** output format"* engage both skills. `cdc-review` produces findings on the Go error chain; `cdc-err` produces the envelope. They cross-link in the report so an operator can see how a swallowed cause upstream becomes a missing `detail` field downstream.

## Why not bundle this with `pr-review`?

`pr-review` is a generalist with a finite review budget. Error handling is a deep, language-specific topic where most reviews degenerate into "use Result" or "wrap with `%w`" without the surrounding context (when to recover, when to abort, what to log, what to emit). `cdc-review` exists to be the specialist `pr-review` calls when an edit touches error code, and to decline cleanly when the work is generic style or test coverage.

## Why not bundle this with `test-architect`?

`test-architect` covers the *coverage* of error paths. `cdc-review` covers their *shape*. The two cooperate — a `cdc-review` finding will sometimes say "and `test-architect` should add a regression test here" — but they don't overlap.

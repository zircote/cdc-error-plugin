---
id: explanation-dual-consumer-problem
type: semantic
created: 2026-05-21T10:47:00-04:00
diataxis_type: explanation
title: Why CLI errors are a dual-consumer problem
---

# Why CLI errors are a dual-consumer problem

CLIs used to have one audience: a human at a terminal. Pretty colors, prose, the occasional traceback — all good.

That assumption is gone. Every CLI now has a second audience: an LLM agent reading the same `stderr` through a `tool_result` payload. The two audiences want incompatible things, and CLIs that optimize for one make the other expensive or unsafe.

## The two audiences in one table

|  | Human at terminal | LLM agent running the tool |
|---|---|---|
| Wants | Rich, colorized prose | Structured fields it can branch on |
| Reads | `stderr` directly | `tool_result` payload |
| Costs of getting it wrong | Frustration, lost trust | Wasted tokens, abandoned recoverable work, wrong fixes auto-applied |

## Why one-audience output fails the other

**Pretty-only output fails agents.** A 5KB Python traceback is roughly 1,600 tokens of `tool_result` budget, almost none of it actionable. A `429 Too Many Requests` without `retry_after` makes the agent give up on a perfectly recoverable error. An unmarked "did you mean…" suggestion gets applied verbatim by an agent that doesn't know it's a guess.

**JSON-only output fails humans.** Operators scanning logs for the actual problem don't want to `jq` every line. Color and layout are not decoration; they are the thing.

The solution isn't to pick one. It's to ship both, and let the consumer choose.

## The dual-consumer pattern

1. **Two renderers, one error event.** The underlying error object carries the structured fields. Two renderers translate it: one to pretty prose, one to a `problem+json` envelope. Neither is derived from the other.
2. **Renderer selection.** Default to TTY-detection on stderr. Allow `--format=json|pretty` to override.
3. **One canonical schema.** RFC 9457 `application/problem+json` is the envelope. Five standard fields (`type`, `title`, `status`, `detail`, `instance`) plus three mandatory agent extensions (`retry_after`, `suggested_fix`, `code_actions[]`).
4. **Applicability markers on suggestions.** Borrowed from `rustc`: every `suggested_fix` carries `machine_applicable` / `maybe_incorrect` / `has_placeholders` / `unspecified`. Without this, agents can't safely auto-apply fixes.

## Why this is `cdc-err`'s entire scope

`cdc-err` exists to enforce this pattern in CLI tools. It does not address:

- application logging (different concern),
- HTTP API errors consumed by a browser (use RFC 9457 directly; the dual-consumer framing is CLI-specific),
- frontend exception display,
- internal telemetry shape.

For source-code error propagation behind the CLI — swallowed exceptions, `unwrap()` in library code, lost cause chains — see [why the plugin ships three sibling skills](skill-cooperation.md).

## Source

The dual-consumer framing comes from the post that originated this skill: <https://zircote.com/blog/2026/04/cli-error-messages-are-a-dual-consumer-problem/>. The skill is faithful to that source; anything outside it is marked out of scope rather than invented.

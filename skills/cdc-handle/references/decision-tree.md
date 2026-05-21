# The Decision Tree

This reference covers Decide mode — given a parsed envelope, pick one of four decisions.

```
retry        — re-invoke the same tool with the same arguments, after a delay
apply_fix    — perform the action described by suggested_fix or a code_actions[] entry
handback     — emit a structured summary to the human and stop
abort        — give up on the task entirely (rare; only for unambiguous terminal errors)
```

## The lattice

Evaluate the conditions top to bottom. The **first** one that matches wins.

| # | Condition | Decision | Reason |
|---|---|---|---|
| 1 | Envelope is malformed (any item from `parsing.md` malformation catalogue) | `handback` | Acting on inconsistent envelopes is the failure mode cdc-err exists to prevent. |
| 2 | `status_class == success` | (skill should not have been invoked) | Parse mode + advise caller. |
| 3 | `status_class == transient` AND `retry_after` is a non-null number ≥ 0 | `retry` | Producer told you when to retry. Trust it. |
| 4 | `status_class == transient` AND `retry_after` is `null` or `absent` | `handback` | Do not guess a backoff. Without `retry_after`, an agent inventing a delay is exactly what cdc-err was designed to prevent. |
| 5 | `status_class == client_error` AND `code_actions[]` includes a `retry` kind | `handback` | A `retry` action on a client error is contradictory — it should be `apply_fix` or `exec`. Flag the malformation. |
| 6 | `status_class == client_error` AND `suggested_fix.applicability == machine_applicable` | `apply_fix` | Producer affirmed the fix is safe to auto-apply. |
| 7 | `status_class == client_error` AND `suggested_fix.applicability` is `maybe_incorrect` / `has_placeholders` / `unspecified` / absent | `handback` | The applicability gate has not been cleared. |
| 8 | `status_class == client_error` AND `suggested_fix` is null AND `code_actions[]` is empty | `handback` | Producer gave no recovery hint. |
| 9 | `status_class == server_error` AND `code_actions[]` includes a `retry` kind with `applicability == machine_applicable` AND `retry_after` present | `retry` | Producer explicitly asked for a retry on a non-transient server error. Honor it. |
| 10 | `status_class == server_error` (otherwise) | `handback` | Treat 5xx as terminal unless the producer explicitly opts in to retry. |
| 11 | `status_class == other` | `handback` | Malformed status. |

## When `abort` instead of `handback`?

`abort` is rare and only safe in narrow conditions:
- The agent has no human-in-the-loop (e.g. a fully autonomous batch job).
- AND the envelope is `client_error` with `suggested_fix == null` and `code_actions[] == []`.
- AND the agent's parent workflow has declared the task non-retryable.

Otherwise default to `handback`. The cost of an unwanted abort (a task quietly given up) outweighs the cost of a handback (a human reads a summary).

## Idempotence and retry safety

`retry` is only safe when the underlying operation is idempotent. The envelope itself does not declare idempotence — that is a property of the *call*, not the *error*. The agent's caller (the harness or workflow) must know whether the tool is idempotent. If it does not know, downgrade `retry` to `handback`.

A reasonable heuristic: tools that are read-only or that have a stable resource identifier (e.g. `kubectl apply`, `aws s3 cp`, HTTP `GET` / `PUT` / `DELETE`) are idempotent; tools that mutate without a stable key (e.g. `curl -X POST /transfer`) are not.

## Multiple code_actions

When `code_actions[]` has more than one entry, the agent picks **one** to act on. Selection rules in order:

1. Prefer `machine_applicable` over any other applicability.
2. Prefer `retry` kind when `status_class == transient`.
3. Prefer `exec` / `edit` over `unknown` kinds when `status_class == client_error`.
4. If still tied, take the lowest-index entry.

The unchosen entries appear in the `handback` plan as alternative actions — the human can pick a different one.

## How the decision feeds the plan

| Decision | Plan shape |
|---|---|
| `retry` | One step: `{ kind: "retry", action: "<original tool call>", args: { delay_s: retry_after }, applicability_gate: "machine_applicable" }` |
| `apply_fix` | One step per `code_actions[]` entry chosen, or one step from `suggested_fix` if no code_actions. See `code-actions.md`. |
| `handback` | One step: `{ kind: "handback", action: "emit_handback_to_user", args: { envelope: <original>, reason: <one-line reason from the lattice row> } }` |
| `abort` | One step: `{ kind: "abort", action: "stop_task", args: { reason: <one-line>} }` |

Every plan ends with the decision JSON; Act mode produces the steps, Decide mode produces the same JSON with `plan: []`.

## Worked example: 503 with retry_after present

Envelope (post-parse):
- `status_class: transient`, `status: 503`
- `retry_after: 30`
- `suggested_fix: null`
- `code_actions: []`

Walk the lattice: row 1 (no malformations) → row 3 matches (transient + retry_after non-null).

Decision: `retry`. Plan:

```json
{
  "decision": "retry",
  "reason": "503 with retry_after=30 — producer explicitly authorized a retry after 30 seconds.",
  "plan": [
    {
      "step": 1,
      "kind": "retry",
      "applicability_gate": "machine_applicable",
      "action": "rerun_failed_tool",
      "args": { "delay_s": 30 },
      "on_failure": "handback"
    }
  ],
  "envelope_refs": { "status": 503, "type": "...", "retry_after": 30 }
}
```

## Worked example: 422 with has_placeholders fix

Envelope (post-parse):
- `status_class: client_error`, `status: 422`
- `suggested_fix: { value: "pip install <name>", applicability: "has_placeholders" }`
- `code_actions: [{ kind: "exec", applicability: "has_placeholders", args: { command: "pip install <name>" } }]`

Walk: row 1 (no malformations), row 6 fails (`applicability != machine_applicable`), row 7 matches.

Decision: `handback`. Plan emits a handback step that names the proposed action and its placeholder, asking the user to fill in `<name>` or to confirm a substitution.

```json
{
  "decision": "handback",
  "reason": "422 client error with suggested_fix carrying has_placeholders applicability — the fix template needs human values before it can be applied.",
  "plan": [
    {
      "step": 1,
      "kind": "handback",
      "applicability_gate": "has_placeholders",
      "action": "emit_handback_to_user",
      "args": {
        "proposed_action": "exec: pip install <name>",
        "placeholders": ["<name>"],
        "applicability": "has_placeholders"
      },
      "on_failure": "abort"
    }
  ],
  "envelope_refs": { "status": 422, "type": "...", "retry_after": null }
}
```

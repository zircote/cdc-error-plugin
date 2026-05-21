# Parsing a cdc-err Envelope

This reference covers Parse mode — extracting structure from a `tool_result` payload that is (or claims to be) a `cdc-err`-shaped RFC 9457 envelope.

## The minimum recognition contract

A payload is treated as a cdc-err envelope when **all** of the following hold:

1. The body parses as JSON.
2. The body contains at least four of the five RFC 9457 standard members: `type`, `title`, `status`, `detail`, `instance`.
3. The body contains at least one of the three mandatory `cdc-err` extensions: `retry_after`, `suggested_fix`, `code_actions[]`.

If only (1) and (2) hold and **no** extensions are present, the envelope is RFC 9457 but not cdc-err-shaped — handle Parse but route Decide to `handback`.

If (1) fails, the payload is unstructured — say so explicitly and stop.

## The Parse-mode output shape

```json
{
  "mode": "parse",
  "envelope_kind": "cdc-err | rfc9457-only | malformed | non-json",
  "status_class": "transient | client_error | server_error | success | other",
  "type_uri": "<value of type, or null>",
  "title": "<value of title, or null>",
  "detail": "<value of detail, or null>",
  "instance": "<value of instance, or null>",
  "extensions": {
    "retry_after": <number | iso8601 string | null | "absent">,
    "suggested_fix": {
      "value": "<string or null>",
      "applicability": "machine_applicable | maybe_incorrect | has_placeholders | unspecified | absent"
    },
    "code_actions": [
      {
        "index": <int>,
        "kind": "<verbatim kind string>",
        "title": "<title or null>",
        "applicability": "<one of the four markers, or absent>",
        "args_present": <bool>
      }
    ]
  },
  "malformations": [
    "<one line per malformation, see catalogue below>"
  ]
}
```

`absent` is distinct from `null`. cdc-err requires producers to emit each extension key even when the value is `null` (so the agent doesn't have to guess whether the field was forgotten or intentionally empty). `absent` therefore signals a deviation from the contract.

## Status classes

| Class | HTTP status codes | What it means for the agent |
|---|---|---|
| `success` | 2xx | The caller shouldn't have invoked this skill. Say so and stop. |
| `client_error` | 4xx (except 429) | Do not retry. Look at `suggested_fix` and `code_actions[]`. |
| `transient` | 408, 425, 429, 502, 503, 504 | Possibly retryable. `retry_after` is load-bearing. |
| `server_error` | 5xx (except the transients above) | Treat as terminal unless `code_actions[]` includes a retry. |
| `other` | Anything outside 1xx–5xx, or `status` missing | Malformed. Add a malformation note. |

cdc-err does **not** require any particular `status` value — it only requires that one be present and that it inform the agent's retry decision. The classes above are conventions consistent with RFC 9110 § 15.

## Malformation catalogue

Emit one entry in `malformations[]` for each that applies. Each is grounds for routing Decide mode to `handback`.

1. **`type` missing or not a URI.** RFC 9457 requires a URI; cdc-err requires a stable URI policy.
2. **`status` missing or not an integer.** The retry decision cannot be made without it.
3. **`title` or `detail` missing.** The human-readable summary fields. Their absence makes handback to a human impossible.
4. **All three extensions absent.** Cannot be cdc-err-shaped.
5. **`status` is in the transient class but `retry_after` is `absent`.** The exact failure mode cdc-err exists to prevent.
6. **`suggested_fix` present but applicability is `absent`.** Treat the fix as `maybe_incorrect`; record the malformation.
7. **`code_actions[]` entry with `kind` `absent` or empty.** Cannot dispatch on it.
8. **`status` and `type` contradict.** E.g. `status: 200` with `type: .../rate-limit-exceeded`. This is a producer bug; do not infer one over the other.
9. **`instance` missing.** Less critical, but flag — the human-readable handback lacks a request ID.
10. **`retry_after` is a negative number, or a malformed ISO-8601 string.** Treat as `absent`.

## Examples

### Well-formed cdc-err envelope

```json
{
  "type": "https://api.example.com/errors/rate-limit-exceeded",
  "title": "Rate limit exceeded",
  "status": 429,
  "detail": "Exceeded 100 requests/minute on /v1/embed.",
  "instance": "urn:request:2026-05-21T14:22:10Z-req-abc123",
  "retry_after": 180,
  "suggested_fix": {
    "value": "Wait 180 seconds before retrying.",
    "applicability": "machine_applicable"
  },
  "code_actions": [
    {
      "kind": "retry",
      "title": "Retry after 180 seconds",
      "applicability": "machine_applicable",
      "args": { "delay_s": 180 }
    }
  ]
}
```

Parse output: `envelope_kind: cdc-err`, `status_class: transient`, no malformations.

### Transient with absent retry_after (the canonical failure)

```json
{
  "type": "https://api.example.com/errors/upstream-unavailable",
  "title": "Upstream unavailable",
  "status": 503,
  "detail": "The embedding service is unreachable.",
  "instance": "urn:request:2026-05-21T14:23:00Z-req-def456"
}
```

Parse output: `envelope_kind: rfc9457-only`, `status_class: transient`, `malformations: ["transient status without retry_after", "all three mandatory extensions absent"]`.

### Client error with has_placeholders fix

```json
{
  "type": "https://cli.example.com/errors/missing-dep",
  "title": "Missing dependency",
  "status": 422,
  "detail": "Package <name> is required for this command.",
  "instance": "urn:request:2026-05-21T14:25:00Z-req-ghi789",
  "retry_after": null,
  "suggested_fix": {
    "value": "Run: pip install <name>",
    "applicability": "has_placeholders"
  },
  "code_actions": [
    {
      "kind": "exec",
      "title": "Install missing package",
      "applicability": "has_placeholders",
      "args": { "command": "pip install <name>" }
    }
  ]
}
```

Parse output: `envelope_kind: cdc-err`, `status_class: client_error`, no malformations. (Note: `has_placeholders` is not a malformation — it's a valid applicability marker. The marker tells the agent the fix isn't ready to apply as-is.)

## What Parse mode does not do

- It does not propose an action.
- It does not retry.
- It does not edit any file.
- It does not call the failing tool again.

Parse mode is purely structural. If the caller wants a decision or an action, they invoke Decide or Act after seeing the Parse output.

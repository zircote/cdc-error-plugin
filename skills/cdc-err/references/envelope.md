# RFC 9457 Problem Details Envelope — Full Spec for CLI Errors

Source: https://zircote.com/blog/2026/04/cli-error-messages-are-a-dual-consumer-problem/

## Media Type

`application/problem+json`

## Standard Members (RFC 9457)

| Member     | Type        | Meaning                                                                 |
|------------|-------------|-------------------------------------------------------------------------|
| `type`     | URI string  | A URI reference identifying the problem type. Default: `about:blank`.   |
| `title`    | string      | Short, human-readable summary of the problem type. Stable per `type`.   |
| `status`   | number      | JSON number mapping to exit code or status class.                       |
| `detail`   | string      | Human-readable explanation specific to **this occurrence**.             |
| `instance` | URI string  | URI reference identifying the specific occurrence (e.g. a `urn:` form). |

`title` is keyed to `type` and should not vary per occurrence. `detail` carries the per-occurrence specifics.

## Mandatory Extensions for Agent Consumers

These three are required by the post — without them the envelope still serves humans but fails the LLM consumer.

| Extension       | Type                     | Meaning                                                                                  |
|-----------------|--------------------------|------------------------------------------------------------------------------------------|
| `retry_after`   | number (delta-seconds) or RFC 3339 timestamp | When the operation may safely be retried. Mandatory for any transient/rate-limit class. |
| `suggested_fix` | string or object         | Free-text or structured description of the recovery action.                              |
| `code_actions[]`| array of LSP CodeAction-shaped objects | Structured edits the agent can apply directly.                              |

### Applicability Markers (rustc precedent, required by the post)

Every `suggested_fix` and every entry in `code_actions[]` carries one of:

- `machine_applicable` — agent can edit and retry without human confirmation.
- `maybe_incorrect` — agent must escalate to a human.
- `has_placeholders` — fix contains slots the agent must fill, lower confidence.
- `unspecified` — applicability unknown; treat as `maybe_incorrect`.

Without these markers, agents will apply plausible-looking but wrong fixes. The post is emphatic about this.

## Optional Extensions

| Extension           | Type     | Meaning                                                              |
|---------------------|----------|----------------------------------------------------------------------|
| `exit_code`         | number   | The process exit code emitted alongside the error.                   |
| `libraries_missing` | string[] | Names of system libraries the failure references (e.g. `["ssl"]`).   |
| `docs_url`          | URI      | Stable URL to long-form documentation for this problem type.         |

## Exit Code / Status Mapping

The post gives one explicit mapping (`status: 429 → exit_code: 2` for rate limits) but does **not** prescribe a complete table. Out of scope: defining your own table is the CLI author's responsibility. The envelope must carry both `status` and `exit_code` so each consumer reads the field native to it.

## Format Selection

A dual-format CLI **ships both renderings in the same binary** and selects between them by:

1. Explicit flag — `--format=json` or `--format=pretty`.
2. TTY detection — if stdout/stderr is not a TTY, default to JSON.

Both signals must be honored. The human rendering "remains as lush as it ever was."

## Versioning of `type` URIs

The post requires a stable URI policy. Two acceptable shapes:

- Embed the version in the URI: `https://docs.example.com/cli/errors/v2/linker-missing-library`.
- Keep the URI stable and track a changelog at the documentation endpoint.

Either is fine. The author must **commit** to one and not silently change `type` semantics.

## Worked Example — Linker Error

```json
{
  "type": "https://docs.example.com/cli/errors/linker-missing-library",
  "title": "Linker cannot find a required system library",
  "status": 471,
  "detail": "ld failed: cannot find -lssl. OpenSSL headers not installed.",
  "instance": "urn:build:f9e4c2b1-a3d5-4e7f-9b8c-1d2e3f4a5b6c",
  "exit_code": 1,
  "libraries_missing": ["ssl", "crypto"],
  "suggested_fix": "Install libssl-dev (Debian/Ubuntu) or openssl-devel (RHEL/Fedora)",
  "docs_url": "https://docs.example.com/cli/errors/linker-missing-library"
}
```

## Worked Example — Rate Limit (canonical)

```json
{
  "type": "https://api.example.com/errors/rate-limit-exceeded",
  "title": "Rate limit exceeded",
  "status": 429,
  "detail": "Exceeded rate limit for this endpoint.",
  "instance": "urn:request:2026-04-15T14:22:10Z-req-abc123",
  "exit_code": 2,
  "retry_after": 180,
  "suggested_fix": "Wait 180 seconds before retrying.",
  "docs_url": "https://api.example.com/docs/rate-limits"
}
```

`retry_after` is what unlocks recovery for the agent consumer. Omit it and "some model responses will abandon the task entirely."

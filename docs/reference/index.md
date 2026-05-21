---
diataxis_type: reference
title: Reference index
---

# Reference index

Information-oriented material. Each section points to the canonical file inside the skill it belongs to.

## `cdc-err` — CLI output

| Topic | File |
|---|---|
| RFC 9457 envelope spec + 3 mandatory agent extensions + applicability markers | [`skills/cdc-err/references/envelope.md`](../../skills/cdc-err/references/envelope.md) |
| Per-language idioms (Rust/miette, Python/Rich, Go/Cobra) | [`skills/cdc-err/references/languages.md`](../../skills/cdc-err/references/languages.md) |
| 7-section audit checklist for error output | [`skills/cdc-err/references/review-checklist.md`](../../skills/cdc-err/references/review-checklist.md) |
| Eval set (12 evals, 50 expectations, 63 deterministic checks) | [`skills/cdc-err/evals/evals.json`](../../skills/cdc-err/evals/evals.json) |

## `cdc-handle` — agent-side envelope interpretation

| Topic | File |
|---|---|
| Recognizing cdc-err envelopes in `tool_result`, status classes, malformation catalogue | [`skills/cdc-handle/references/parsing.md`](../../skills/cdc-handle/references/parsing.md) |
| Retry / apply_fix / handback / abort decision lattice | [`skills/cdc-handle/references/decision-tree.md`](../../skills/cdc-handle/references/decision-tree.md) |
| Applicability gate (machine_applicable / maybe_incorrect / has_placeholders / unspecified) | [`skills/cdc-handle/references/applicability.md`](../../skills/cdc-handle/references/applicability.md) |
| Translating `code_actions[]` entries into executable plan steps | [`skills/cdc-handle/references/code-actions.md`](../../skills/cdc-handle/references/code-actions.md) |
| Eval set (15 evals, 69 expectations, 76 deterministic checks) | [`skills/cdc-handle/evals/evals.json`](../../skills/cdc-handle/evals/evals.json) |

## `cdc-review` — source-code error handling

| Topic | File |
|---|---|
| 8-section code-review checklist | [`skills/cdc-review/references/checklist.md`](../../skills/cdc-review/references/checklist.md) |
| Severity taxonomy (`must-fix` / `should-fix` / `nit` / `question` / `praise`) with error-handling criteria | [`skills/cdc-review/references/severity.md`](../../skills/cdc-review/references/severity.md) |
| YAML work-plan format for large refactors | [`skills/cdc-review/references/workplan.md`](../../skills/cdc-review/references/workplan.md) |
| Language references | [`skills/cdc-review/references/languages/`](../../skills/cdc-review/references/languages/) |
| Eval set (12 evals, 53 expectations, 53 deterministic checks) | [`skills/cdc-review/evals/evals.json`](../../skills/cdc-review/evals/evals.json) |

### Language references

| Language | File |
|---|---|
| Rust — `thiserror`/`anyhow`, `?`, panic boundaries | [`languages/rust.md`](../../skills/cdc-review/references/languages/rust.md) |
| Go — `errors.Is`/`As`, `%w`, `errors.Join` | [`languages/go.md`](../../skills/cdc-review/references/languages/go.md) |
| Python — PEP 3134 `raise from`, `with`, `contextlib.suppress` | [`languages/python.md`](../../skills/cdc-review/references/languages/python.md) |
| TypeScript — `Error.cause`, `neverthrow`, async pitfalls | [`languages/typescript.md`](../../skills/cdc-review/references/languages/typescript.md) |
| Java — try-with-resources, checked vs unchecked | [`languages/java.md`](../../skills/cdc-review/references/languages/java.md) |

## Plugin manifest

| Field | Value |
|---|---|
| `name` | `error-handling` |
| `version` | `0.2.0` |
| `license` | MIT |
| Manifest | [`.claude-plugin/plugin.json`](../../.claude-plugin/plugin.json) |

## Trigger phrases

`cdc-err` activates on phrases including: *"review my CLI error messages"*, *"my CLI dumps a traceback when the API rate-limits us"*, *"add a compliant error to my Cobra CLI"*, *"problem+json"*, *"dual-consumer errors"*, *"RFC 9457"*.

`cdc-review` activates on phrases including: *"review my error handling"*, *"audit error code"*, *"find error-handling bugs"*, *"review my Result usage"*, *"audit my errors.Is calls"*, *"check my try/except"*, *"panic in library"*, *"swallowed error"*, *"lost cause chain"*. Also fires automatically when a PR / file edit touches error-emission, propagation, recovery, or cleanup code.

`cdc-handle` activates on phrases including: *"interpret this error"*, *"should I retry this CLI error"*, *"the tool_result has problem+json"*, *"can I auto-apply this suggested_fix"*, *"what does this retry_after mean"*, *"the CLI returned an error envelope"*. Also auto-triggers whenever a `tool_result` payload contains the cdc-err contract fields (`retry_after`, `suggested_fix`, `code_actions[]`) or an `application/problem+json` body.

## Envelope at a glance

The RFC 9457 + agent-extensions envelope every `cdc-err` output emits:

```json
{
  "type": "https://example.com/problems/rate-limited",
  "title": "Rate limited",
  "status": 429,
  "detail": "Upstream API rate-limited this request.",
  "instance": "/requests/abc123",
  "retry_after": 30,
  "suggested_fix": "Wait and retry, or reduce request rate.",
  "applicability": "machine_applicable",
  "code_actions": [
    { "title": "Retry after 30s", "kind": "retry", "args": { "delay_s": 30 } }
  ]
}
```

Full schema and field semantics: [`skills/cdc-err/references/envelope.md`](../../skills/cdc-err/references/envelope.md).

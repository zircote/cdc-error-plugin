---
name: cdc-handle
description: Interpret and act on RFC 9457 problem+json error envelopes that a CLI (typically one shaped by sibling skill cdc-err) returns to an LLM agent in a tool_result. Three modes — Parse the envelope, Decide retry vs apply-fix vs handback vs abort, Act by emitting a concrete executable action plan. Always gates suggested_fix and code_actions[] on applicability markers (machine_applicable / maybe_incorrect / has_placeholders / unspecified) and treats missing/unspecified applicability as maybe_incorrect (safe default — surface to human, never auto-apply). Use this skill whenever a tool_result payload contains an application/problem+json envelope, fields like retry_after / suggested_fix / code_actions[], or status / title / type / detail / instance combinations characteristic of RFC 9457. Also use on explicit phrasing — "interpret this error", "should I retry this CLI error", "the CLI returned problem+json", "what does this retry_after mean", "can I auto-apply this suggested_fix", "the tool_result has an error envelope", "how should the agent handle this error". Defers envelope schema and producer-side questions to cdc-err, and source-code propagation questions to cdc-review.
argument-hint: "[parse|decide|act] [<envelope-json-or-tool_result-excerpt>]"
allowed-tools: "Read, Write, Bash, Grep, Glob"
license: MIT
---

# Handling cdc-err-Shaped Error Envelopes

This skill is the **consumer-side** counterpart to `cdc-err`. Where `cdc-err` codifies how a CLI should emit a dual-format error, this skill codifies how an LLM agent should **interpret and act on** that error when it lands in a `tool_result` payload.

It is intentionally narrow: it handles envelopes shaped per the RFC 9457 Problem Details contract plus the three mandatory agent extensions (`retry_after`, `suggested_fix`, `code_actions[]`) and the applicability markers (`machine_applicable` / `maybe_incorrect` / `has_placeholders` / `unspecified`). Envelopes that omit the mandatory extensions are flagged as malformed and routed to a human; envelopes from non-cdc-err producers are out of scope (defer to a generic RFC 9457 reader).

## The three modes

The skill always executes in one mode. Pick from the input shape:

| Mode | Decision rule | Output |
|---|---|---|
| **Parse** | Caller hands you a `tool_result` payload or raw envelope and asks "what does this mean?" / "what's in this error?" / "explain this envelope". | Structured summary of the envelope: status class, transient vs terminal, what each agent extension contains, which applicability marker each fix carries. No action proposed. |
| **Decide** | Caller asks "what should I do with this?" / "should I retry?" / "is this recoverable?" — they want the choice between retry / apply-fix / handback / abort. | A single decision (`retry` / `apply_fix` / `handback` / `abort`) with the reasoning, the inputs the decision rests on, and the gating that produced it. No code is run. |
| **Act** | Caller asks "do it" / "apply the fix" / "execute the action" / "carry out the recovery". | An **executable action plan**: a sequence of concrete tool calls or commands the agent should run next, with applicability gating applied. Plan items have `kind`, `tool`/`command`, `args`, `applicability_gate`, and an `on_failure` fallback. |

Three modes are mutually exclusive. If the input is ambiguous, default to **Decide** — it's the safest middle: a recommendation with reasoning, no action taken.

Once the mode is clear, load only the reference you need:

| File | Load when… |
|---|---|
| `references/parsing.md` | Parse mode — exact field semantics, recognizing malformed envelopes, status-class classification. |
| `references/decision-tree.md` | Decide mode — the retry/apply/handback/abort lattice. |
| `references/applicability.md` | Any time a `suggested_fix` or `code_actions[]` entry needs to be acted on. |
| `references/code-actions.md` | Act mode — how to translate `code_actions[]` entries into concrete tool calls. |

## The output contract: an executable action plan

Act mode always produces a JSON block of this exact shape:

```json
{
  "decision": "retry | apply_fix | handback | abort",
  "reason": "one-sentence explanation tying the decision to specific envelope fields",
  "plan": [
    {
      "step": 1,
      "kind": "<code_action kind verbatim, or 'retry' / 'handback' / 'abort'>",
      "applicability_gate": "machine_applicable | maybe_incorrect | has_placeholders | unspecified",
      "action": "concrete tool call or shell command the agent will run",
      "args": { },
      "on_failure": "what to do if this step fails — usually 'handback'"
    }
  ],
  "envelope_refs": {
    "status": <int>,
    "type": "<type URI>",
    "retry_after": <number | null>
  }
}
```

Decide mode produces the same JSON with `plan` empty and `decision` filled. Parse mode produces a different summary shape (see `references/parsing.md`) with no `decision`.

## The non-negotiables

The skill's correctness rests on four rules. None of them are stylistic:

1. **Applicability gates every action that touches user-owned state.** If `suggested_fix` or a `code_actions[]` entry has applicability `maybe_incorrect`, `has_placeholders`, `unspecified`, **or applicability is absent**, the agent does not apply it autonomously. The plan emits a `handback` step describing the proposed fix and the reason for not applying it. This is the failure mode `cdc-err` exists to prevent — never undermine it on the consumer side.

2. **`retry_after` is load-bearing.** When `status` is in the transient class (429, 502, 503, 504) and `retry_after` is present and non-null, emit a `retry` step with `args.delay_s = retry_after`. When `status` is transient and `retry_after` is **absent or null**, emit `handback` — the envelope is malformed; **do not guess a backoff delay**. Inventing a retry interval when the server did not provide one is exactly the failure mode `cdc-err` exists to prevent. Use the phrase "do not guess" (or "cannot guess") explicitly in the reason field so the human reading the handback understands why no retry was attempted.

3. **Unknown `code_actions[].kind` does not block the response.** The skill recognizes `kind` as an opaque string. For kinds it does not recognize, it still applies the applicability gate, and if applicability allows, emits a `handback` step describing the action shape and asking the human to confirm. The skill never invents a new kind's semantics.

4. **A malformed envelope is itself a finding.** Missing required RFC 9457 members (`type`, `title`, `status`, `detail`, `instance`), missing all three mandatory extensions, or a `status` that contradicts `type` (e.g. a 4xx error from a `rate-limit-exceeded` type) all produce a `handback` plan with the malformation noted. The skill does not act on inconsistent envelopes.

## The decision tree, in one paragraph

Read `status`. If 2xx (no error — caller shouldn't have invoked this skill), say so and stop. If 4xx **client error**: do not retry; look at `suggested_fix` and `code_actions[]`. Apply the applicability gate: `machine_applicable` → `apply_fix`; otherwise → `handback`. If 5xx server error or 429: look at `retry_after`. If present and the request is idempotent (or `code_actions[]` includes a `retry` kind), `decision = retry`. If absent or `null`, `decision = handback`. The full table including edge cases is in `references/decision-tree.md`.

## Sibling skill handoffs

- **`cdc-err`** owns the producer side. If the caller asks "what fields should this envelope contain?" or "what should the CLI emit for a rate-limit error?", name `cdc-err` and stop. **Do not enumerate the envelope schema, the mandatory extensions, or the applicability markers as part of the handoff.** The caller needs to consult `cdc-err`, not receive a summary from `cdc-handle`. A handoff response should be two to three sentences: state the question is producer-side, name `cdc-err`, and stop.
- **`cdc-review`** owns source-code propagation. If the caller asks "the CLI that's emitting these envelopes has bad error propagation upstream — review the code", name `cdc-review` and stop.

Cross-link in both directions. The three skills together cover producer (`cdc-err`), propagation (`cdc-review`), and consumer (`cdc-handle`). Each declines cleanly outside its lane. A clean decline means a brief redirect — not a helpful orientation that partially answers the out-of-scope question.

## Out of scope

- Envelopes that are not RFC 9457 problem+json (raw stderr, ad-hoc JSON error shapes, plaintext tracebacks). Say so explicitly; defer to the user or a more general error-reading skill.
- Designing the envelope (that's `cdc-err`).
- Reviewing the CLI's internal error-propagation code (that's `cdc-review`).
- Generic HTTP retry policies for browser frontends (the dual-consumer framing is CLI-specific).
- Token-cost accounting for response bodies (the math lives in `cdc-err`).
- Adding telemetry or logging for handled errors.

## How to recognize a cdc-err envelope in a tool_result

Heuristics, in order of confidence:

1. The `tool_result` body's media type is `application/problem+json`.
2. The body parses as JSON and contains **all three** mandatory extensions (`retry_after`, `suggested_fix`, `code_actions`) — even when their values are `null`. cdc-err requires explicit nulls; a producer that emits all three keys is almost certainly cdc-err-shaped.
3. The body contains `type`, `title`, `status`, `detail` (any three of four) and at least one of the mandatory extensions.

If only (3) matches and the extensions are mostly absent, the envelope is RFC 9457 but not cdc-err-shaped. Process Parse mode, but in Decide mode return `handback` with a note that the producer is not following the cdc-err contract.

## Source

The contract this skill consumes is defined in [`../cdc-err/references/envelope.md`](../cdc-err/references/envelope.md), itself drawn from [Allen Walker's *"CLI Error Messages Are a Dual-Consumer Problem"*](https://zircote.com/blog/2026/04/cli-error-messages-are-a-dual-consumer-problem/) and RFC 9457 § 3 (Problem Details for HTTP APIs).

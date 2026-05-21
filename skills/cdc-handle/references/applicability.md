# Applicability Markers — The Auto-Apply Gate

Applicability markers are the single most important decision the agent makes when handling a cdc-err envelope. They determine whether the agent may take action that touches user-owned state autonomously, or whether the human is required to approve.

## The four markers

Borrowed verbatim from the rustc diagnostic guide and adopted by cdc-err.

| Marker | Meaning | Agent behavior |
|---|---|---|
| `machine_applicable` | The producer has affirmed this fix is correct and safe to apply without human review. | **Auto-apply permitted.** Emit an `apply_fix` plan step. |
| `maybe_incorrect` | The fix is plausible but may be wrong for the user's platform / version / context. | **Do not auto-apply.** Emit a `handback` plan step that names the proposed fix and asks the human to confirm. |
| `has_placeholders` | The fix contains tokens like `<name>` / `${VAR}` that must be filled in by a human or by a more-informed caller. | **Do not auto-apply.** Do not substitute guessed values. Emit a `handback` plan step that enumerates the placeholders. |
| `unspecified` | The producer did not commit to any applicability. | **Treat as `maybe_incorrect`.** Do not auto-apply; handback. |

## Missing applicability is treated as `unspecified`

When a `suggested_fix` or a `code_actions[]` entry omits the `applicability` field entirely, the skill treats it as `unspecified` — which means `maybe_incorrect` — which means handback. **Missing is never a license to act.** Record the absence as a **malformation** in the Parse output. Use the word "malformation" explicitly; absent applicability is a malformation of the envelope shape, not merely a producer-side defect or an omission.

The reasoning: cdc-err's producer contract makes applicability mandatory. A producer that omits it is either (a) buggy or (b) deliberately ambiguous. Either way the safest read is "do not auto-apply." This is the cheap-cost / high-cost asymmetry — a wrongly-applied fix costs the user real work; a handback costs the user one extra prompt.

## Why this rule is non-negotiable

> A 5KB Python traceback is roughly 1,600 tokens of tool_result budget, almost none of it actionable. A `429 Too Many Requests` without `retry_after` makes the agent give up. **An unmarked "did you mean…" suggestion gets applied verbatim by an agent that doesn't know it's a guess.**
> — *"CLI Error Messages Are a Dual-Consumer Problem"*

cdc-err exists in part to remove the third failure mode. The consumer side (this skill) is the other half of that contract. If applicability gating is loose here, the producer's investment is wasted.

## The gate, expressed as pseudocode

```python
def may_auto_apply(suggested_fix_or_code_action) -> bool:
    appl = suggested_fix_or_code_action.get("applicability")
    if appl is None:
        return False  # absent → treat as unspecified → no
    if appl == "machine_applicable":
        return True
    return False  # maybe_incorrect, has_placeholders, unspecified
```

A two-line gate. The rest of this skill's complexity exists so that the two lines do not get bypassed.

## Examples by marker

### `machine_applicable` — auto-apply OK

```json
{
  "suggested_fix": {
    "value": "Set environment variable EMBED_API_KEY (from your dashboard).",
    "applicability": "machine_applicable"
  }
}
```

Plan emits `apply_fix` with `args.action = "set_env"`. Note: even `machine_applicable` actions still respect the caller's allow-list — the harness is the final authority on what tool calls are permitted. Applicability is *necessary* for auto-apply, not *sufficient*.

### `maybe_incorrect` — handback

```json
{
  "suggested_fix": {
    "value": "Try `--retry-count=3`.",
    "applicability": "maybe_incorrect"
  }
}
```

Plan emits `handback` describing the proposed flag and asking the user to confirm.

### `has_placeholders` — handback with enumeration

```json
{
  "suggested_fix": {
    "value": "Run: pip install <package-name>",
    "applicability": "has_placeholders"
  }
}
```

Plan emits `handback` and enumerates `<package-name>` as the placeholder to be filled.

### `unspecified` — same handback as `maybe_incorrect`

```json
{
  "suggested_fix": {
    "value": "Re-run with --verbose to see more.",
    "applicability": "unspecified"
  }
}
```

Plan emits `handback`. The handback prose may suggest `--verbose` but does not run it.

### Applicability absent — record malformation, treat as `unspecified`

```json
{
  "suggested_fix": {
    "value": "Add --retry to your invocation."
  }
}
```

Parse output adds a malformation: `"suggested_fix present but applicability absent"`. Decide mode treats this as `unspecified` → `handback`.

## Edge case: applicability on `code_actions[]` entries

Each entry in `code_actions[]` carries its own applicability. The skill never aggregates them. If `code_actions[0].applicability == machine_applicable` and `code_actions[1].applicability == has_placeholders`, the plan may include `code_actions[0]` as `apply_fix` and surface `code_actions[1]` as an alternative under the handback.

## Edge case: contradiction between `suggested_fix.applicability` and `code_actions[N].applicability`

cdc-err allows either or both to be present. If both are present and they disagree (e.g. `suggested_fix.applicability == machine_applicable` but `code_actions[0].applicability == maybe_incorrect`), the **stricter marker wins**: take the most-cautious marker as the effective gate. Use that exact phrase — "the stricter marker wins" — when explaining this precedence rule. Reason: the producer's contract is to mark applicability honestly; the agent's contract is to err on the side of caution when the producer is internally inconsistent. Do not auto-apply when markers conflict.

## What this skill never does

- It never **upgrades** an applicability marker. `maybe_incorrect` cannot become `machine_applicable` because the agent has read the fix and thinks it looks fine. The producer is the only authority.
- It never **downgrades** an applicability marker either. If the producer said `machine_applicable`, the skill does not override that by scanning the fix string for tokens that *look* like placeholders (e.g. `<name>`, `${VAR}`, `<from-vault>`). Token-shape inspection is not a substitute for the marker. Only the `has_placeholders` marker — set by the producer — triggers placeholder treatment. A `machine_applicable` fix that contains angle-bracket text is a producer choice (perhaps the angle brackets are literal, perhaps they're an intentional convention) and the skill must honor it.
- It never **strips** a placeholder and applies the resulting string. `<name>` is a marker, not a placeholder syntax to be normalized.
- It never **auto-fills** placeholders from context. Even when the agent can guess (e.g. it knows the missing package name from earlier conversation), the gate forbids applying it autonomously — the handback names the placeholder and asks.

### Why this rule is important (and easy to get wrong)

The skill must resist a tempting heuristic: "this `machine_applicable` fix contains `<from-vault>` so it's *probably* a placeholder, so I should treat it as `has_placeholders`." That heuristic looks safety-positive but is actually a contract violation: it lets the consumer second-guess the producer, which destroys the producer's incentive to mark applicability honestly. If the producer is wrong about `machine_applicable`, that is a producer bug to file — not a consumer override.

The only legitimate consumer-side conservatism is the *stricter-wins* rule when `suggested_fix.applicability` and a `code_actions[]` entry's applicability **explicitly disagree**. That rule compares two *producer-supplied* markers; it does not invent a third marker by inspecting fix content.

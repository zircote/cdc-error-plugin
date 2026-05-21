# Executing `code_actions[]`

This reference covers Act mode — translating a `code_actions[]` entry into a concrete tool call the agent will run next.

`cdc-err` says each `code_actions[]` entry is an "LSP CodeAction-shaped object" but does not enumerate kinds. This skill treats `kind` as an opaque string and handles it by shape.

## Required fields the skill expects on each entry

```json
{
  "kind": "<string>",
  "title": "<human-readable label>",
  "applicability": "<one of the four markers, or absent>",
  "args": { /* kind-specific */ }
}
```

`args` is `kind`-specific. The skill does not validate `args` schemas; it forwards them verbatim into the plan step and lets the executing layer interpret them.

## How a `code_actions[]` entry becomes a plan step

```json
{
  "step": <N>,
  "kind": "<copied verbatim from code_action.kind>",
  "applicability_gate": "<copied verbatim from code_action.applicability, or 'unspecified' if absent>",
  "action": "<the concrete tool name or shell command the agent should run>",
  "args": <copied from code_action.args>,
  "on_failure": "handback"
}
```

The `action` field is the only place the skill makes a translation from `kind` to a real-world call. Translation is by recognition; unknown kinds fall through to `handback`.

## Translation table (recognized kinds)

These are kinds the skill recognizes by convention. Unknown kinds are handed back rather than guessed.

| `kind` | Typical meaning | Concrete `action` (when gate clears) |
|---|---|---|
| `retry` | Re-invoke the failing tool after `args.delay_s` seconds. | `rerun_failed_tool` (the harness re-issues the original call). |
| `edit` | Apply a textual edit to a file. `args` mirrors LSP `WorkspaceEdit` (`uri`, `range`, `newText`) or carries a unified diff. | `edit_file` (Edit tool) when shape matches; `handback` otherwise. |
| `exec` | Run a shell command. `args.command` is the command string. | `run_command` (Bash tool). |
| `env-set` | Set or export an environment variable. `args.name` and `args.value`. | `set_env` (harness-dependent). |
| `install-dep` | Install a missing dependency. `args.package`, `args.manager`. | `run_command` with the manager's install verb. |
| `file-create` | Create a new file. `args.path`, `args.content`. | `write_file` (Write tool). |
| `config-set` | Update a configuration value. `args.path` (e.g. JSON pointer), `args.value`. | `handback` by default — config edits cross too many trust boundaries to apply autonomously without explicit `machine_applicable` and harness allow-listing. |

For any kind not in this table, emit `handback` describing the action shape:

```json
{
  "step": 1,
  "kind": "<unknown kind verbatim>",
  "applicability_gate": "<from code_action>",
  "action": "emit_handback_to_user",
  "args": {
    "unknown_kind": "<kind>",
    "original_code_action": <verbatim entry>,
    "note": "The cdc-err producer used a kind this skill does not recognize. The original entry is included verbatim so the human can decide."
  },
  "on_failure": "abort"
}
```

## The gate is applied **before** translation

Translation happens only when `may_auto_apply(code_action) == True`. If the gate fails, the plan emits `handback` with the proposed action described in prose, never a callable step that touches state.

This is the load-bearing rule: a `retry` action with `applicability: maybe_incorrect` does not become a plan step that retries. It becomes a handback that says "the producer suggested a retry but did not mark it machine_applicable; do you want to retry?"

## Selecting among multiple entries

When `code_actions[]` has more than one entry, pick one per the rules in `decision-tree.md`:

1. Highest applicability (machine_applicable > others).
2. Most appropriate kind for the status class.
3. Lowest index when still tied.

The unchosen entries appear in the plan as `alternatives` so the human can pick a different one during handback. Do not chain multiple entries autonomously without explicit user consent — a sequence of even individually-safe actions is a different trust boundary.

## When the same envelope has both `suggested_fix` and `code_actions[]`

cdc-err allows both. They are not redundant — `suggested_fix` is prose for the human renderer and a fallback for agents that don't speak `code_actions[]`; `code_actions[]` is structured and machine-actionable.

Act mode prefers `code_actions[]` when it is non-empty and at least one entry has `applicability == machine_applicable`. Otherwise `suggested_fix` is the source for the handback prose.

## Worked example: `retry` action, applicability cleared

Envelope (post-parse):
- `status: 429`, `retry_after: 60`
- `code_actions: [{ kind: "retry", applicability: "machine_applicable", args: { delay_s: 60 } }]`

Plan:

```json
{
  "decision": "retry",
  "reason": "429 with retry_after=60 and a machine_applicable retry code_action — auto-retry permitted.",
  "plan": [
    {
      "step": 1,
      "kind": "retry",
      "applicability_gate": "machine_applicable",
      "action": "rerun_failed_tool",
      "args": { "delay_s": 60 },
      "on_failure": "handback"
    }
  ],
  "envelope_refs": { "status": 429, "type": "...", "retry_after": 60 }
}
```

## Worked example: `edit` action, applicability `machine_applicable`

Envelope (post-parse):
- `status: 422`
- `code_actions: [{ kind: "edit", applicability: "machine_applicable", title: "Add missing comma", args: { uri: "file:///workspace/app.py", range: {...}, newText: "," } }]`

Plan:

```json
{
  "decision": "apply_fix",
  "reason": "422 with a machine_applicable edit code_action — the producer affirmed the edit is correct.",
  "plan": [
    {
      "step": 1,
      "kind": "edit",
      "applicability_gate": "machine_applicable",
      "action": "edit_file",
      "args": { "uri": "file:///workspace/app.py", "range": {...}, "newText": "," },
      "on_failure": "handback"
    }
  ],
  "envelope_refs": { "status": 422, "type": "...", "retry_after": null }
}
```

The harness applies the edit via its Edit tool. If the edit fails (file gone, range stale), `on_failure` routes to handback — the agent does not retry an edit blindly.

## Worked example: unknown kind

Envelope (post-parse):
- `status: 500`
- `code_actions: [{ kind: "rotate-credentials", applicability: "machine_applicable", args: { vault: "prod" } }]`

The kind is not in the translation table. Plan:

```json
{
  "decision": "handback",
  "reason": "500 with a code_action kind 'rotate-credentials' not recognized by this skill — handing back to the human with the original entry verbatim.",
  "plan": [
    {
      "step": 1,
      "kind": "rotate-credentials",
      "applicability_gate": "machine_applicable",
      "action": "emit_handback_to_user",
      "args": {
        "unknown_kind": "rotate-credentials",
        "original_code_action": {
          "kind": "rotate-credentials",
          "applicability": "machine_applicable",
          "args": { "vault": "prod" }
        }
      },
      "on_failure": "abort"
    }
  ],
  "envelope_refs": { "status": 500, "type": "...", "retry_after": null }
}
```

Even with `machine_applicable`, the skill refuses to invent semantics. The human gets the verbatim entry and decides.

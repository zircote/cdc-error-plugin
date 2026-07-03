---
id: how-to-add-cdc-err-to-cli
type: procedural
created: 2026-05-21T10:47:00-04:00
diataxis_type: how-to
title: Add cdc-err dual-format output to an existing CLI
---

# Add cdc-err dual-format output to an existing CLI

You already have a CLI that writes errors to stderr. You want LLM agents calling it to get a `problem+json` envelope while humans keep the pretty output.

## Steps

1. **Decide on the renderer-selection contract.** The recommended default:
   - `--format=json` → JSON envelope to stderr
   - `--format=pretty` → human prose to stderr
   - no flag → auto-detect (`isatty(stderr)` → pretty; else json)

2. **Trigger the skill.** In Claude Code, in your project:

   ```
   add a problem+json envelope to <command> in <file>
   ```

   `cdc-err` will produce both renderers and the selection logic for your language.

3. **Required envelope fields.** Every emitted error must include:
   - RFC 9457 standard: `type`, `title`, `status`, `detail`, `instance`
   - Agent extensions: `retry_after`, `suggested_fix`, `code_actions[]`
   - Applicability marker on `suggested_fix`: one of `machine_applicable`, `maybe_incorrect`, `has_placeholders`, `unspecified`

4. **Keep the human renderer.** Do not delete your existing pretty output. The skill preserves it and adds JSON alongside.

5. **Test both formats.**

   ```bash
   yourcli broken-command --format=pretty 2>&1 >/dev/null
   yourcli broken-command --format=json    2>&1 >/dev/null | jq .
   ```

   The pretty form should still be colorized and readable. The JSON should parse and validate against the envelope schema in [reference/envelope](../reference/index.md#envelope-at-a-glance).

## Common pitfalls

- **Don't strip ANSI from the json path.** They are separate renderers, not a transformation.
- **`retry_after` is mandatory even when not applicable** — set it to `null` and explain why in `detail`. Don't omit the field.
- **`suggested_fix` without applicability marker is worse than no fix at all.** Agents will apply `unspecified` fixes verbatim.

## When to call the sibling skill

If during the migration you find the underlying code is also propagating errors badly (swallowed exceptions, lost cause chains, `unwrap()` in library code), `cdc-err` will name `cdc-review` and hand off. Run:

```
review error propagation in <file>
```

separately, then return to the envelope work.

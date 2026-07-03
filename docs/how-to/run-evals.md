---
id: how-to-run-and-improve-skill-evals
type: procedural
created: 2026-05-21T10:47:00-04:00
diataxis_type: how-to
title: Run and improve the skill evals
---

# Run and improve the skill evals

All three skills carry eval sets used by the `autoresearch` plugin to iterate on skill quality.

## Inspect the eval set

```bash
jq '.evals | length' skills/cdc-err/evals/evals.json     # 12
jq '.evals | length' skills/cdc-review/evals/evals.json  # 12
jq '.evals | length' skills/cdc-handle/evals/evals.json  # 15
```

Expectation and deterministic-check counts:

```bash
jq '.evals | length,
    ([.[].expectations | length] | add),
    ([.[].deterministic_checks // [] | length] | add)' \
   skills/cdc-err/evals/evals.json
```

## Audit / repair the eval set

```
/autoresearch --eval-doctor skills/cdc-err
/autoresearch --eval-doctor skills/cdc-review
/autoresearch --eval-doctor skills/cdc-handle
```

This finds presence-only assertions, ungrep-able expectations, and discriminating-power gaps.

## Run the improvement loop

```
/autoresearch skills/cdc-err
/autoresearch skills/cdc-review
/autoresearch skills/cdc-handle
```

Modify → evaluate → keep-or-discard, until convergence. The repo's `.gitignore` already excludes the transient `*-autoresearch/`, `*-workspace/`, and `*-dashboard.html` artifacts the loop produces.

## Quality target

Keep the deterministic-check ratio ≥50% — that is, at least as many regex/grep checks as LLM expectations. Below that, the loop has no real signal to optimize against.

Current ratios:

- `cdc-err`: 63 deterministic / 113 total = 55.8%
- `cdc-review`: 53 deterministic / 106 total = 50.0%
- `cdc-handle`: 76 deterministic / 145 total = 52.4%

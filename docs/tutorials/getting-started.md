---
id: tutorial-getting-started-error-handling
type: procedural
created: 2026-05-21T10:47:00-04:00
diataxis_type: tutorial
title: Get started with the error-handling plugin
---

# Get started with the error-handling plugin

This tutorial walks you through installing the plugin and seeing both skills trigger on a real piece of code. You will:

1. Install the plugin from the marketplace.
2. Ask Claude to review a deliberately broken CLI error path — `cdc-review` fires.
3. Ask Claude to add a dual-format error to the same code — `cdc-err` fires.
4. See the two skills hand off cleanly.

You need: Claude Code installed, a terminal, and ~10 minutes.

## 1. Install the plugin

In Claude Code:

```
/plugin marketplace add zircote/cdc-error-plugin
/plugin install error-handling@cdc-errors
```

When the picker shows `cdc-err`, `cdc-review`, and `cdc-handle` under `Available skills`, the plugin is loaded.

## 2. Create the example code

Make a throwaway directory and paste this Python in `app.py`:

```python
import sys, requests

def fetch(url):
    try:
        r = requests.get(url, timeout=5)
        r.raise_for_status()
        return r.json()
    except Exception:
        pass  # swallow

if __name__ == "__main__":
    data = fetch(sys.argv[1])
    print(data)
```

The bug is the bare `except: pass` — it swallows the cause chain and the program then crashes with a confusing `TypeError` because `data` is `None`.

## 3. Trigger `cdc-review`

In Claude Code, in that directory:

```
review the error handling in app.py
```

`cdc-review` activates. You should see:

- a classified finding tagged `[must-fix]` for the swallowed exception,
- a before/after diff that re-raises with cause preservation,
- at least one `[praise]` finding (the timeout was set correctly).

This is the source-code-quality side of the plugin.

## 4. Trigger `cdc-err`

Now ask for the output format:

```
make app.py emit a problem+json envelope to stderr when the fetch fails
```

`cdc-err` activates. You should see:

- a JSON envelope with the five RFC 9457 fields (`type`, `title`, `status`, `detail`, `instance`),
- the three mandatory agent extensions (`retry_after`, `suggested_fix`, `code_actions[]`),
- a `--format=json|pretty` flag with TTY auto-detection.

This is the CLI-output side.

## 5. See the handoff

Ask a question that crosses both:

```
review app.py's error output AND propagation
```

Both skills engage, each declining the other's scope and naming the sibling. `cdc-review` covers propagation; `cdc-err` covers the envelope.

## Next steps

- Read [Why CLI errors are a dual-consumer problem](../explanation/dual-consumer.md) for the rationale.
- See [How to add cdc-err to an existing CLI](../how-to/add-cdc-err-to-cli.md) for a real-project recipe.
- Browse the [reference index](../reference/index.md) for envelope fields, severity taxonomy, and language-specific patterns.

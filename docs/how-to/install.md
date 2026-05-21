---
diataxis_type: how-to
title: Install the error-handling plugin
---

# Install the error-handling plugin

Three install paths, ordered from most to least common.

## From the marketplace

```
/plugin install zircote/error-handling
```

Verify both skills loaded:

```
/plugin list
```

You should see `error-handling` with `cdc-err` and `cdc-review` underneath.

## From a local checkout (development)

Clone the repo, then point Claude Code at it via a project-level marketplace in `.claude/settings.json`:

```json
{
  "plugins": {
    "marketplaces": [
      { "path": "/absolute/path/to/error-handling" }
    ]
  }
}
```

Restart Claude Code. Skills auto-discover from `skills/`.

## From a personal marketplace

If you maintain your own marketplace pointing at this repo:

```
/plugin install error-handling
```

## Verify

Open a project with any code in it and type:

```
review error handling in this file
```

If `cdc-review` activates, you're set. If not, check:

- `/plugin list` shows `error-handling` enabled.
- Restart Claude Code if the plugin was installed in this session.
- The trigger phrase matches one in the [skill description](../reference/index.md#trigger-phrases).

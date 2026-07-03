---
id: how-to-install-error-handling-plugin
type: procedural
created: 2026-05-21T10:47:00-04:00
diataxis_type: how-to
title: Install the error-handling plugin
---

# Install the error-handling plugin

This repository is itself a single-plugin marketplace: `.claude-plugin/marketplace.json`
lists one plugin, `error-handling`, sourced from the same repo. Two install paths,
ordered from most to least common.

## From GitHub (recommended)

```
/plugin marketplace add zircote/cdc-error-handling
/plugin install error-handling@error-handling
```

Verify all three skills loaded:

```
/plugin list
```

You should see `error-handling` with `cdc-err`, `cdc-review`, and `cdc-handle` underneath.

## From a local checkout (development)

Clone the repo, then add it as a local marketplace:

```
/plugin marketplace add /absolute/path/to/cdc-error-handling
/plugin install error-handling@error-handling
```

To make this automatic for a project (so teammates are prompted to install it),
add it to `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "error-handling": {
      "source": {
        "source": "github",
        "repo": "zircote/cdc-error-handling"
      }
    }
  }
}
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

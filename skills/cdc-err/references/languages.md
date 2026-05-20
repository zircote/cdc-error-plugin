# Language-Specific Implementation Patterns

Source: https://zircote.com/blog/2026/04/cli-error-messages-are-a-dual-consumer-problem/

The post names three ecosystems explicitly. Other languages (Node, Ruby, Java, C#, shell) are **out of scope** for this skill — apply the same envelope and dual-renderer pattern with that ecosystem's idiomatic tools.

## Rust — miette + thiserror

The post: *"miette extends `std::error::Error` with a `Diagnostic` trait"* and ships `JSONReportHandler` plus `NarratableReportHandler` for dual rendering.

```rust
use miette::{Diagnostic, NamedSource, SourceSpan};
use thiserror::Error;

#[derive(Error, Diagnostic, Debug)]
#[error("linking with cc failed")]
#[diagnostic(
    code(rustc::linker_error),
    help("install libssl-dev and libcrypto, then re-run"),
    url("https://docs.example.com/cli/errors/linker-missing-library")
)]
pub struct LinkerError {
    #[label("ld cannot find -lssl")]
    pub ssl_span: SourceSpan,
    #[label("ld cannot find -lcrypto")]
    pub crypto_span: SourceSpan,
}

// Select renderer based on --format / TTY:
fn install_renderer(json: bool) {
    if json {
        miette::set_hook(Box::new(|_| Box::new(miette::JSONReportHandler::new()))).unwrap();
    } else {
        miette::set_hook(Box::new(|_| Box::new(miette::GraphicalReportHandler::new()))).unwrap();
    }
}
```

Add the RFC 9457 extensions (`retry_after`, `suggested_fix`, `code_actions`) as serde fields on a wrapper struct that wraps the miette `JSONReportHandler` output, or emit the envelope directly with `serde_json` for error classes where a richer agent contract matters.

## Python — Rich + Click/Typer

The post: *"Rich serves the same role"* as miette, used with Click or Typer.

```python
import json, sys
from dataclasses import dataclass, asdict
from rich.console import Console

@dataclass
class ProblemDetails:
    type: str
    title: str
    status: int
    detail: str
    instance: str
    exit_code: int
    retry_after: int | None = None
    suggested_fix: str | None = None
    docs_url: str | None = None

def emit(err: ProblemDetails, fmt: str | None = None) -> None:
    use_json = (fmt == "json") or (fmt is None and not sys.stderr.isatty())
    if use_json:
        sys.stderr.write(json.dumps(asdict(err), separators=(",", ":")) + "\n")
    else:
        console = Console(stderr=True)
        console.print(f"[red bold]{err.title}[/]: {err.detail}")
        if err.suggested_fix:
            console.print(f"[yellow]fix:[/] {err.suggested_fix}")
        if err.retry_after is not None:
            console.print(f"[dim]retry after {err.retry_after}s[/]")
    sys.exit(err.exit_code)
```

The media type for the JSON form is `application/problem+json`; if writing to a file or HTTP response set that header.

## Go — Cobra + encoding/json

The post: use *"Cobra plus any `encoding/json`-based emitter"*.

```go
package clierr

import (
	"encoding/json"
	"fmt"
	"os"

	"github.com/mattn/go-isatty"
)

type Problem struct {
	Type          string   `json:"type"`
	Title         string   `json:"title"`
	Status        int      `json:"status"`
	Detail        string   `json:"detail"`
	Instance      string   `json:"instance"`
	ExitCode      int      `json:"exit_code,omitempty"`
	RetryAfter    *int     `json:"retry_after,omitempty"`
	SuggestedFix  string   `json:"suggested_fix,omitempty"`
	DocsURL       string   `json:"docs_url,omitempty"`
	CodeActions   []Action `json:"code_actions,omitempty"`
}

type Action struct {
	Title         string `json:"title"`
	Edit          string `json:"edit"`
	Applicability string `json:"applicability"` // machine_applicable | maybe_incorrect | has_placeholders | unspecified
}

func Emit(p Problem, format string) {
	useJSON := format == "json" || (format == "" && !isatty.IsTerminal(os.Stderr.Fd()))
	if useJSON {
		_ = json.NewEncoder(os.Stderr).Encode(p)
	} else {
		fmt.Fprintf(os.Stderr, "%s: %s\n", p.Title, p.Detail)
		if p.SuggestedFix != "" {
			fmt.Fprintf(os.Stderr, "  fix: %s\n", p.SuggestedFix)
		}
		if p.RetryAfter != nil {
			fmt.Fprintf(os.Stderr, "  retry after: %ds\n", *p.RetryAfter)
		}
	}
	if p.ExitCode != 0 {
		os.Exit(p.ExitCode)
	}
}
```

Bind a persistent `--format` flag at the root Cobra command so every subcommand picks the same renderer.

## Other Languages

**(Out of scope for this skill in the cited post.)** The post does not name idiomatic libraries for Node, Ruby, Java, C#, or shell. The pattern still applies — pick the ecosystem's pretty-printer for humans, `application/problem+json` for agents, select by `--format` or TTY — but specific library choices are the author's call.

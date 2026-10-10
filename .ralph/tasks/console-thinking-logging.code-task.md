---
status: in-progress
created: 2026-10-19
started: 2026-10-19
completed:
---
# Code Task: Clear Styled Console Logging for Backend Thinking Blocks

## Overview

Requested by the user (2026-10-19): the installed ralph does not visibly show
OMP thinking logs, and the console thinking rendering should be improved to
show all details with clear logging, color, and style.

Repro (installed `ralph 2.10.1`, `ralph run --no-tui`, `backend: omp`,
`show_thinking: true`, fake `omp` emitting pi-family NDJSON):

```
^[[mLet me reason about this in several^[[0m^[[m chunks to verify streaming works.^[[0mDone: LOOP_COMPLETE
```

Thinking streams inline with the assistant answer on the same line: no label,
no boundary, near-invisible color difference, chunks interleaved with the
answer text.

## Changes

### 1. Distinct, labeled thinking blocks in console output

Console renderers must wrap thinking in a visually distinct block:

- **Block open** (first thinking delta of a block): newline + dimmed colored
  `[thinking]` label, body follows on the same line.
- **Body**: dimmed dark grey, streamed in full (no truncation, all chunks).
- **Block close** (next text/tool/error output, or end of stream): newline +
  dimmed grey dashed separator, so the assistant answer starts on a fresh
  line.

State (in-block flag) is owned by the executor scope and passed to the
per-line handlers, since handlers are constructed per stream line.

- **`crates/ralph-adapters/src/cli_executor.rs`** — `WriterStreamHandler`
  (headless `ralph run --no-tui` path): open/body/close rendering with a
  `thinking` block state; state owned in `execute()`; block force-closed at
  stream end.
- **`crates/ralph-adapters/src/stream_handler.rs`** — `ConsoleStreamHandler`
  (PTY console fallback in `loop_runner::execute_pty`): same rendering
  treatment via the shared helper.
- Shared SGR helpers (dim attribute, cyan label, dark-grey body/separator)
  live next to the handlers in `stream_handler.rs`; no new dependency.

TUI rendering is unchanged (thinking already renders as dimmed blocks in the
TUI widget via `RpcEvent::ThinkingDelta`).

## Tests

- `cli_executor` — thinking then text: labeled block, dim SGR, full content,
  separator before the answer, answer on its own line, thinking excluded
  from `extracted_text`.
- `cli_executor` — thinking-only stream (no trailing text): block closed at
  finalization, nothing dangling.
- Existing `test_execute_pi_stream_thinking_console_flag` /
  `test_execute_claude_stream_thinking_console_flag` /
  `pty_executor_integration` console-flag tests updated to the new rendering
  where they asserted the old bare-dim SGR shape.

## Verification

- `cargo test -p ralph-adapters` — scoped, green.
- `cargo test --all --no-fail-fast` — full suite green (flake-free run).
- `cargo fmt --all -- --check` + `cargo clippy -p ralph-adapters -p ralph-cli --all-targets -- -D warnings`.
- Install: `cargo install --path crates/ralph-cli --force`; re-run the fake-`omp`
  `ralph run --no-tui` repro and observe the labeled, styled block in output.

---
status: current
created: 2026-10-19
related:
  - console-thinking-logging (task: .ralph/tasks/console-thinking-logging.code-task.md)
---

# Spec: Clear Styled Console Logging for Backend Thinking Blocks

## Context

The installed ralph does not visibly show OMP (oh-my-pi) thinking logs in
console mode. Repro (installed `ralph 2.10.1`, `ralph run --no-tui`,
`backend: omp`, `cli.show_thinking: true`, fake `omp` emitting pi-family
NDJSON):

```
^[[mLet me reason about this in several^[[0m^[[m chunks to verify streaming works.^[[0mDone: LOOP_COMPLETE
```

Thinking deltas stream inline with the assistant answer on the same line: no
label, no block boundary, near-invisible color difference, chunks fused with
the answer text. Root cause: `WriterStreamHandler::on_thinking`
(`crates/ralph-adapters/src/cli_executor.rs`) wraps each delta in a bare
dark-grey SGR pair with no label or separator, and the handler is
constructed per stream line, so no block state survives across chunks.

Approved scope: user request (2026-10-19) — show thinking with clear
logging, color, and style, and show all details (full content, no
truncation). Console rendering only; TUI rendering already displays
thinking as dimmed blocks and is out of scope.

## Behavior

Console renderers (`WriterStreamHandler` for headless `ralph run --no-tui`,
`ConsoleStreamHandler` for the PTY console fallback in
`loop_runner::execute_pty`) render thinking as a distinct block:

- **Open** (first thinking delta of a block): newline + dimmed cyan
  `[thinking]` label (matching the `[Tool]`/`[Result]`/`[Error]` bracket
  convention in the same file), body follows inline.
- **Body**: dimmed dark grey, streamed in full — every chunk, no
  truncation.
- **Close** (next text/tool/error output, or end of stream): dimmed dark
  grey separator line (`└` + 44 `─`), so the assistant answer starts on a
  fresh line.

Block open/close state (`ThinkingBlock`) is owned by the executor scope and
handed to the per-line handlers, since console handlers are constructed per
stream line. The shared renderer lives in
`crates/ralph-adapters/src/stream_handler.rs`; no new dependencies (crossterm
`SetAttribute(Dim)`, `Color::Cyan`, `Color::DarkGrey`).

Backend logs: OMP writes human log lines outside the NDJSON protocol
(plain lines on stdout, plus all stderr). Both must reach the console
regardless of `verbose`: non-NDJSON stdout lines are routed through the
new `StreamHandler::on_log` (default no-op; the console writers render
them as plain lines, closing any open thinking block first), and OMP
stderr lines are always shown with a `[stderr]` prefix (other backends
keep the verbose gate). Log lines never enter `extracted_text`, are
still counted as malformed lines for the protocol-mismatch check, and
remain in the raw accumulated stream (`result.output`) as before.

Backend errors: a `turn_end` whose message carries an `errorMessage`
(OMP's `stopReason:"error"` records, e.g. a provider HTTP 400) is
surfaced through the existing `StreamHandler::on_error` path so the
failure is visible on the console instead of ending as a silent empty
iteration. The error text never enters `extracted_text` and does not
affect the protocol-mismatch check; session `is_error` is unchanged
(a non-zero backend exit already marks it).

Unchanged: `extracted_text` (thinking never enters it), the
`verbose || show_thinking` gates, TUI `RpcEvent::ThinkingDelta` path,
`PrettyStreamHandler`, `QuietStreamHandler`, the pi-family/Claude parsers.

## Acceptance

1. Console output with `show_thinking` on: labeled `[thinking]` block, dim
   styling (SGR dim attribute present), full thinking content, separator
   before the assistant answer; thinking excluded from `extracted_text`.
2. Thinking-only stream (no trailing text): block closed at stream
   finalization — nothing dangling, no unclosed color state.
3. Without the console flag / verbose: thinking remains hidden from the
   writer (existing behavior preserved).
4. `cargo test -p ralph-adapters` green (incl. updated
   `test_execute_pi_stream_thinking_console_flag`,
   `test_execute_claude_stream_thinking_console_flag`, new
   thinking-only-closes-at-end test); `cargo test --all --no-fail-fast`
   green serially (`-- --test-threads=1`; parallel runs hit a pre-existing
   `loop_runner` flake family — see task file); `cargo fmt --all -- --check` and
   `cargo clippy -p ralph-adapters -p ralph-cli --all-targets -- -D warnings
   -A clippy::double-must-use -A clippy::assert-is-empty -A
   clippy::manual_is_variant_and` clean (the three `-A` allowances skip
   pre-existing lints in dependency crates, verified present on the clean
   `edc2b32` baseline).
5. Reinstalled binary: `cargo install --path crates/ralph-cli --force`;
   fake-`omp` `ralph run --no-tui` repro shows the labeled, styled block.
6. With `backend: omp`: non-NDJSON stdout log lines and stderr log lines
   appear in the console output without verbose; neither enters
   `extracted_text`.
7. OMP stream ending in `stopReason:"error"` with an `errorMessage`:
   the message is shown via the error path, stays out of
   `extracted_text`, and the stream still finalizes normally.

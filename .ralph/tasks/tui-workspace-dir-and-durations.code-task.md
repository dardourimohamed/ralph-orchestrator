---
status: completed
created: 2026-09-25
started: 2026-09-25
completed: 2026-09-25
---
# Code Task: TUI Workspace Dir Display + Hours/Days in Elapsed Times

## Overview

Two display changes to the Ralph TUI (ratatui):

1. Show the current workspace directory in the header status bar, with `~` home shortcut when under `$HOME`, else full path.
2. Show hours and days in the total elapsed time (footer) and the current iteration elapsed time (header).

## Changes

### 1. `~`-shortened workspace directory in header

- **`crates/ralph-tui/src/widgets/header.rs`** — reuses existing `TuiState::export_workspace_root` (defaulted from `RALPH_WORKSPACE_ROOT` / `.ralph` discovery, synced from `loop.started` in RPC mode) instead of new state. Adds `home_dir()` (`HOME` with `USERPROFILE` fallback, cached in a `LazyLock`, matching the `config_resolution::home_dir_from_env` pattern) and `shorten_dir()` (home prefix → `~`). Renders ` | ~/projects/foo @ git:main` as a combined location block, hidden below `WIDTH_SHOW_BRANCH` (65 cols), ellipsis-truncated.
- No new `TuiState` fields; `export_workspace_root` is already preserved across `task.start` resets.

### 2. Hours/days in elapsed times

- **`crates/ralph-core/src/text.rs`** — new `format_duration(secs: u64) -> String`: `MM:SS` under 1h, `H:MM:SS` under 1d, `Dd H:MM` beyond (seconds dropped at day scale for TUI compactness). Sub-hour output is byte-identical to the previous inline formatting. Re-exported from `ralph_core`.
- **`crates/ralph-tui/src/widgets/header.rs`** — iteration/wave timer uses `format_duration`.
- **`crates/ralph-tui/src/widgets/footer.rs`** — `Total Time Elapsed:` uses `format_duration`.

## Tests

- `ralph-core` `text::` — duration formatting incl. day-scale arithmetic (97_500s → `1d 3:05`).
- `ralph-tui` header — dir+branch combined, dir alone, hidden on compact terminal, 5 `shorten_dir` unit tests (home prefix, home itself, outside home, no home, `/home/med2` vs `/home/med` non-match), multi-hour and multi-day iteration timers.
- `ralph-tui` footer — multi-hour and multi-day total elapsed.

## Verification

- `cargo test -p ralph-core text::` — 32 passed.
- `cargo test -p ralph-tui --lib` — 254 passed (3 pre-existing header width-calibration tests updated for the new location span with deterministic workspace roots).
- `cargo test` (workspace) — green except pre-existing flaky WIP tests: `ralph-cli` `loop_runner` wave tests and `ralph-adapters` `omp_process_cleanup` under parallel load; both pass in isolation and belong to uncommitted OMP/PTY backend work, not this change.
- Release build + install: `cargo build --release`, `cargo install --path crates/ralph-cli --force` → `ralph` in `~/.cargo/bin`.

## Complexity

Small — additive formatting/display changes, no behavior or protocol changes.

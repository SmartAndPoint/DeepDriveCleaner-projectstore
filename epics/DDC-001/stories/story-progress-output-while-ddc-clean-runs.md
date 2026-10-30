---
type: story
id: "story-progress-output-while-ddc-clean-runs"
epic: "DDC-001"
title: "Progress output while ddc clean runs"
status: in-progress
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-25
external_refs: {}
tags: [cli, ux, idea]
code_refs: ["crates/ddc-core/src/exec/progress.rs", "crates/ddc-core/src/exec/mod.rs", "crates/ddc-core/src/platform/macos/term.rs", "crates/ddc-core/src/platform/macos/mod.rs", "crates/ddc-core/src/lib.rs", "crates/ddc-cli/src/progress.rs", "crates/ddc-cli/src/cmd/clean.rs", "crates/ddc-cli/src/cmd/clean_items.rs", "crates/ddc-cli/tests/clean_cli.rs", "crates/ddc-core/tests/clean_safety.rs", "crates/ddc-core/tests/clean_items.rs", "crates/ddc-core/tests/in_use.rs", "crates/ddc-core/tests/user_data.rs"]
specs: []
started_at: "2026-10-24T20:31:17.888Z"
closed_at: null
plan_updated_at: "2026-10-24T20:31:17.888Z"
---

# Progress output while ddc clean runs

| Field | Value |
|---|---|
| **Epic** | [DDC-001](../epic.md) |
| **Status** | in-progress |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

An idea from the owner's first real `ddc clean` on the reference machine (2026-10-18): after the confirmation (`Clean 2007 items, 100.7 GiB (frees ~88.6 GiB)? [y/N] y`) the terminal shows nothing until every item is done. The scan phase already has a spinner (`progress::with_spinner` over `scan::Progress`); the cleanup phase has none, although it can take minutes in two places:

- before the first item, `exec::execute` lists local snapshots (`tmutil`, up to 30 s), reads free space and walks every selected tree for `.ddc-keep` markers (`keep_markers`, with a throwaway `Progress::default()`): for this run, a walk of 100.7 GiB in 2007 trees;
- then thousands of renames and removals, and native commands such as `go clean -modcache` that may run up to `NATIVE_TIMEOUT` (600 s).

Show progress for both phases: while preparing, the phase and the files walked so far; while cleaning, items processed out of the total, bytes processed so far (the allocated size of the items handled, whatever their outcome: "freed" stays reserved for the free-space difference in the summary), and what is being cleaned now (the category and the path under `~`, or the native command's name).

## Decomposition

- [x] `ddc-core`: `exec/progress.rs`, an execution counter the caller shares with `exec::execute`, as `scan::Progress` is shared with the scanner.
  - The phase: preparing or cleaning.
  - Files walked by the keep-marker walk, against the selected trees' files.
  - Items processed of the total, and bytes processed of the total.
  - The current item: its index, path, category, action and bytes, and the tool's name while its command runs.
- [x] `ddc-core`: `ExecOptions` gains `progress` and `local_snapshots`. Every struct literal becomes `..ExecOptions::default()`, so the next field costs no call-site edits.
- [x] `ddc-core`: `execute` fills the counter. `keep_markers` walks into it instead of a throwaway `Progress::default()`, and `act` names the tool around each `native::run`.
- [x] `ddc-cli`: a pure line formatter (counter snapshot → line) and a progress line on stderr while `execute` runs, only when stderr is a terminal (so a `--yes` run on a terminal shows it too), cleared before the result is printed.
- [x] `ddc-cli`: the terminal clean passes the local-snapshot count it already took before the prompt, so `tmutil` runs once.
- [ ] Owner: the manual check.
- Ctrl-C needs nothing new here. Since story "CLI: clean an explicit item list", a clean installs a SIGINT/SIGTERM handler (`crates/ddc-core/src/sys/signals.rs`) and stops after the current item; the line is cleared before the result in that case too.

## Implementation Plan

Planned 2026-10-25 with the planner. No spec covers this story. The route follows how "ddc scan report" split the scan's progress: the counter in the core (`scan::Progress`), the line in the CLI (`progress::with_spinner`). The core cannot print (`lib.rs` denies `print_stderr`).

1. **`crates/ddc-core/src/exec/progress.rs` (new)**, re-exported from `exec/mod.rs`. It holds:
   - `ExecProgress`: the phase (`AtomicBool`: preparing, then cleaning); `walk: scan::Progress` for the keep-marker walk; `walk_expected`; `items_total`, `items_done`, `bytes_total`, `bytes_done` (`AtomicU64`); `current: Mutex<Option<CurrentItem>>`;
   - `CurrentItem`: 1-based index, path, category, `ActionKind`, bytes, and `tool: Option<&'static str>`;
   - `ExecSnapshot`, built by `snapshot()`.

   Mutators are `pub(crate)`, as `scan::Progress::add` is. Counters use `Relaxed` ordering; the cancel flag stays `SeqCst`.
2. **`ExecOptions`** gains `progress: Option<&ExecProgress>` and `local_snapshots: Option<usize>`, both defaulting to `None`. The seven struct literals become `..ExecOptions::default()`: four in the core tests, one in the core clean-items tests, and the two command call sites.
3. **`execute`:**
   - Totals come from the chosen items. `bytes_total` leaves `native-only` items out (their space is not in the total, safety ADR §5). `walk_expected` is the sum of their `files`.
   - `keep_markers` walks into the shared counter. The counter is cumulative over the per-target walks.
   - The phase turns to cleaning right before the item loop.
   - Each item is started, then done after its journal record. An item left `not-run` by a cancel before it starts is neither started nor counted; one a cancel stops inside `act` (during a tool's location probe) is started but not counted. Either way the count does not jump on a cancel. A journal error clears the current item before it ends the run.
   - `act` sets the tool's name around each `native::run` and clears it after.
   - With `local_snapshots` given, `execute` uses it instead of running `tmutil` again.
4. **CLI `progress.rs`:**
   - The body of `with_spinner` is factored into a private spinner shared by `with_spinner` and a new `with_clean_line(&ExecProgress, home, f)`.
   - A pure `clean_line(&ExecSnapshot, home, width) -> String` formats both phases:
     - preparing: the files checked for keep markers, against the expected count, clamped;
     - cleaning: item `i` of `n`, bytes processed of the total, then the category and the path under `~`, or the tool's name while its command runs.
   - Widths are counted in columns, cautiously: any non-ASCII character as two, so a CJK path or a terminal that draws `·` wide never wraps the line; one column is kept spare.
   - The path goes through `render::tilde`, which escapes control and bidirectional characters, and is shortened in the middle to the terminal's width (`ddc_core::sys::term::stderr_columns`, an `ioctl(TIOCGWINSZ)` read on each tick; 80 columns when unknown), so the line never wraps (`\r\x1b[2K` clears only the last physical line).
   - Writes ignore errors (`let _ = write!(…)`), never `eprint!`, which panics when the terminal is gone.
5. **Call sites:**
   - `cmd/clean.rs` wraps `execute` in `with_clean_line` and passes `progress` and the snapshot count it took before the prompt;
   - `cmd/clean_items.rs` does the same without the snapshot count.

   Both draw the line only when stderr is a terminal.

## Acceptance Criteria

- [ ] **Totals reached.** A core test (`crates/ddc-core/tests/clean_safety.rs`) drives `exec::execute` with fake ports and a shared counter. At the end:
  - items done equal items total, which equals the number selected;
  - bytes done equal bytes total, which equals their bytes, with a `native-only` item left out of both;
  - the same holds when one item is skipped (a `.ddc-keep` added after the scan);
  - after a cancel following the first item, items done stay at 1.
- [ ] **Preparing counts the files.** A test reads the counter at the fake process table's first `processes()` call, which `Guard::new` makes after the marker walk and before the first item. It sees the preparing phase, files walked equal to the selected trees' files and to the expected count, and no item done.
- [ ] **The tool's name.** A test sees the tool's name in the current item while its fake command runs, and, when the item is journaled (still current), no name.
- [ ] **One `tmutil` call.** With `local_snapshots` given, `execute` runs no `tmutil` command, and the report carries the given count (test).
- [ ] **Formatter.** Unit tests of the line formatter cover:
  - the preparing phase, with and without an expected count;
  - the cleaning phase, with a category and a `~/…` path, a control character escaped, a long path shortened in the middle to the width, and a path of wide characters within the width;
  - a tool's name.
- [ ] **No line when piped.** A piped `ddc clean --yes` run (stderr not a terminal) that succeeds prints no progress line. Its stderr holds no `\r` and no erase-line sequence, and no line other than the `ddc: warning:` lines `main.rs` may print at startup in a restricted environment. This is a new assertion in `crates/ddc-cli/tests/clean_cli.rs`.
- [ ] **Manual check** on the reference machine, or under `script -q /dev/null`: the line moves during both phases of a real `ddc clean` and is gone before the result is printed (owner).
- [ ] `make check` passes.

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- **Seams.**
  - `exec::execute(env, catalog, plan, consent, ports, opts)` and `keep_markers` in `crates/ddc-core/src/exec/mod.rs`. `ExecOptions` is the struct "CLI: clean an explicit item list" made for extension.
  - The spinner in `crates/ddc-cli/src/progress.rs`.
  - The call sites in `crates/ddc-cli/src/cmd/clean.rs` and `cmd/clean_items.rs`.
- **The current item needs a lock.** `scan::Progress` is atomics only, but the current item's path and the tool's name need a `Mutex`. Use the poisoning idiom of `keep_markers`.
- **Show the tool's name.** A native command is a single item that can run for minutes; its name is better than a frozen count.
- **One `tmutil` call.** `cmd/clean.rs` already counts local snapshots before the prompt, and `execute` counts them again (up to 30 s). Passing the earlier count in shortens the preparing phase instead of only displaying it.
- **The expected file count is an estimate.** A picked user-data entry's finding has `files: 0`, so the walk can exceed it; clamp on display.
- **Reading the counter mid-run.** The preparing criterion needs a mid-run observation. A fake port read on its first call (the process table) does it without a second thread.
- **The width helper lives in the core's `sys` module.** That is the one module allowed `unsafe` (workspace lint), and `ddc-cli` has `libc` only as a dev-dependency. It reads a width and prints nothing, so "the core never talks to the terminal" still holds; `lib.rs` says so.
- **Ctrl-C during the marker walk.** `keep_markers` checks the cancel flag between targets, not inside one target's walk, so a single large tree is walked to its end before the clean stops. This predates the story. Setting the shared walk counter's cancel from the flag would stop it mid-walk; it is left for "CLI: progress events", which owns the ticker that sees the flag, and which makes a cancelled walk keep its target, as a failed one does.
- **The signal's notice** ("ddc: stopping after the current item") is written raw, so it can appear after a half-drawn progress line. This is cosmetic; the line is redrawn below it and cleared before the result.
- **The JSON events.** The counter is needed beyond the terminal: "CLI: progress events" (DDC-002) emits the same snapshot as `--progress json` and depends on this story. The counter stays in the core and the line in the CLI, so both read one snapshot.

## Dependencies

- [[epics/DDC-001/stories/story-safe-cleanup-ddc-clean|Safe cleanup: ddc clean]]
- Needed by: [[epics/DDC-002/stories/story-cli-progress-events|CLI: progress events]] (DDC-002)

## Attachments

-

---

*Last updated: 2026-10-25*

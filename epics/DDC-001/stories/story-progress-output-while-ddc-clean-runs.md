---
type: story
id: "story-progress-output-while-ddc-clean-runs"
epic: "DDC-001"
title: "Progress output while ddc clean runs"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-18
external_refs: {}
tags: [cli, ux, idea]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Progress output while ddc clean runs

| Field | Value |
|---|---|
| **Epic** | [DDC-001](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

An idea from the owner's first real `ddc clean` on the reference machine (2026-10-18): after the confirmation (`Clean 2007 items, 100.7 GiB (frees ~88.6 GiB)? [y/N] y`) the terminal shows nothing until every item is done. The scan phase already has a spinner (`progress::with_spinner` over `scan::Progress`); the cleanup phase has none, although it can take minutes in two places:

- before the first item, `exec::execute` lists local snapshots (`tmutil`, up to 30 s), reads free space and walks every selected tree for `.ddc-keep` markers (`keep_markers`, with a throwaway `Progress::default()`): for this run, a walk of 100.7 GiB in 2007 trees;
- then thousands of renames and removals, and native commands such as `go clean -modcache` that may run up to `NATIVE_TIMEOUT` (600 s).

Show progress for both phases: while preparing, the phase and the files walked so far; while cleaning, items processed out of the total, bytes processed so far (the allocated size of the items handled, whatever their outcome: "freed" stays reserved for the free-space difference in the summary), and what is being cleaned now (the category and the path under `~`, or the native command's name).

## Decomposition

Rough until the plan gate:

- [ ] `ddc-core`: an execution progress counter the caller shares with `exec::execute`, the way `scan::Progress` is shared with the scanner: the phase (preparing, cleaning), files walked, items processed, bytes processed, and the current item
- [ ] `ddc-core`: `keep_markers` walks with the caller's counter instead of a throwaway `Progress::default()`, so the preparing phase moves
- [ ] `ddc-cli`: a pure line formatter (counter snapshot → line) and a progress line on stderr while `execute` runs, only when stderr is a terminal (so a `--yes` run on a terminal shows it too), cleared before `prompt::print_result`
- Not in this story: stopping on Ctrl-C between items. There is no signal handling today, a handler is new signal-handling code in the core's `sys` module (the only one allowed `unsafe`) or a new dependency, `execute` has no cancel flag, and a native command in the terminal's process group receives SIGINT itself. The journal records only completed actions; an interrupted item leaves a tombstone. SIGTERM handling for a clean is in the DDC-002 story "CLI: clean an explicit item list".

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] A test drives `exec::execute` with fake ports and sees the counter reach the number of selected items and their total bytes, also in a run where one item is skipped
- [ ] A test sees the preparing phase count the files of the selected trees before the first item is processed
- [ ] Unit tests of the line formatter cover the preparing phase, the cleaning phase with a path, and a native command's name
- [ ] A piped `ddc clean --yes` run (stderr not a terminal) that succeeds prints no progress line: stderr holds no `\r` or erase-line sequence and no line other than the `ddc: warning:` lines `main.rs` may print at startup in a restricted environment (a new assertion in `crates/ddc-cli/tests/clean_cli.rs`)
- [ ] Manual check on the reference machine (or under `script -q /dev/null`): the line moves during both phases of a real `ddc clean` and is gone before the result is printed

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- Seams: `exec::execute(env, catalog, plan, consent, ports)` and `keep_markers` in `crates/ddc-core/src/exec/mod.rs`; the spinner in `crates/ddc-cli/src/progress.rs`; the call site in `crates/ddc-cli/src/cmd/clean.rs`.
- `scan::Progress` is atomics only; the current item's path or command name needs a `Mutex<Option<…>>` or similar, so it is not a drop-in copy.
- Changing `execute`'s signature touches every test in `crates/ddc-core/tests/clean_safety.rs`; carrying the counter in `Ports` or an options struct may keep that small.
- A native command is a single item that can run for minutes: show its name rather than a frozen count.
- The preparing phase repeats work: `cmd/clean.rs` already calls `exec::local_snapshots` (up to 30 s) before the prompt, and `execute` calls it again. Passing the earlier result in shortens the phase instead of only displaying it.
- The criterion on the preparing phase needs a mid-run observation: a fake port (the journal or the trasher) holding the counter and reading it on its first call does it without a second thread.
- The counter is needed beyond the terminal: the DDC-002 story "CLI: progress events" emits it as `--progress json` events for the menu bar app, and depends on this story (raised from p3 to p2 for that reason on 2026-10-18). Keep the counter in the core and the stderr line in the CLI, so both read the same snapshot.

## Dependencies

- [[epics/DDC-001/stories/story-safe-cleanup-ddc-clean|Safe cleanup: ddc clean]]
- Needed by: [[epics/DDC-002/stories/story-cli-progress-events|CLI: progress events]] (DDC-002)

## Attachments

-

---

*Last updated: 2026-10-18*

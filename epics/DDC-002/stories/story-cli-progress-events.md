---
type: story
id: "story-cli-progress-events"
epic: "DDC-002"
title: "CLI: progress events"
status: in-progress
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-25
external_refs: {design_canvas: "https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV"}
tags: [cli, contract, json, ux]
code_refs: ["README.md", "app/DeepDriveCleaner/ConfirmSheet.swift", "app/Packages/DDCKit/Sources/DDCKit/CLIRunner.swift", "app/Packages/DDCKit/Sources/DDCKit/CleanProgress.swift", "app/Packages/DDCKit/Sources/DDCKit/CleanReport.swift", "app/Packages/DDCKit/Sources/DDCKit/CleanRun.swift", "app/Packages/DDCKit/Sources/DDCKit/Confirmation.swift", "app/Packages/DDCKit/Sources/DDCKit/Formatting.swift", "app/Packages/DDCKit/Sources/DDCKit/PopoverModel.swift", "app/Packages/DDCKit/Sources/DDCKit/ProgressEvent.swift", "app/Packages/DDCKit/Sources/DDCKit/ReviewModel.swift", "app/Packages/DDCKit/Tests/DDCKitTests/CLIRunnerTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/CleanRunTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ContractTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/FakeCLI.swift", "app/Packages/DDCKit/Tests/DDCKitTests/PopoverModelTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/PopoverSupport.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ProgressTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/RealCLIDryRunTests.swift", "crates/ddc-cli/src/cli.rs", "crates/ddc-cli/src/cmd/clean.rs", "crates/ddc-cli/src/cmd/clean_items.rs", "crates/ddc-cli/src/cmd/scan.rs", "crates/ddc-cli/src/progress.rs", "crates/ddc-cli/tests/clean_items_cli.rs", "crates/ddc-cli/tests/scan_cli.rs", "crates/ddc-core/src/exec/mod.rs", "crates/ddc-core/src/exec/progress.rs", "crates/ddc-core/src/lib.rs", "crates/ddc-core/src/progress.rs", "crates/ddc-core/src/store.rs", "crates/ddc-core/tests/clean_safety.rs", "crates/ddc-core/tests/progress.rs", "docs/progress.schema.json"]
specs: []
started_at: "2026-10-24T20:32:04.261Z"
closed_at: null
plan_updated_at: "2026-10-24T20:32:04.261Z"
---

# CLI: progress events

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The app shows a scan's and a clean's progress from the CLI's machine-readable events. This story adds `--progress json` (§6 item 3 of the ADR "Menu bar app: Swift shell over the ddc CLI").

The clean's counter is built by the DDC-001 story "Progress output while ddc clean runs". This story only emits it as JSON, and cannot start before that story is done.

### Revision after the design review (2026-10-21)

Artboard 11 of the design canvas ("Clean progress") draws the determinate left-to-right bar the owner asked for, in place of the sliding one. Reviewing it fixed what the events must carry.

**The clean's real sequence** is waiting → preparing → cleaning → stopping (`ReviewModel.Phase`); `.preparing` covers the CLI's rescan and then its keep-marker walk.
- Step 1, "Checking again", is that phase, with two counters. The rescan counts files read against the last report's `totals.files`, an estimate if the roots changed. The `keep_markers` walk counts files walked against the listed findings' `files`. Today that walk takes a throwaway `Progress::default()` (`crates/ddc-core/src/exec/mod.rs:445`), so it must report into the clean's progress.
- Step 2 is the item loop.

**Each item event carries:**
- `i` of `n`;
- its path, or the tool's name;
- `action` and `outcome`;
- its `bytes`.

An item's bytes count as processed whatever the outcome; the app adds "· N skipped".

**The bar.**
- **Basis:** bytes processed, with items as the alternative.
- **Item in flight:** shown as a striped segment, because there is no progress inside an item.
- **Native-only item:** takes its share of items, never bytes. Its space is not in the total (safety ADR §5), and the sheet says that what Docker frees inside its disk image is not counted. The bar's byte total therefore leaves native-only items out. Today `Confirmation.bytes` and `selectedReclaimable` include them (`Confirmation.swift:64`, `ReviewModel.swift:333`), so the sheet's header total changes with this story, or the bar never reaches 100 %.

**Stop.**
- Between file items, the clean stops after the current one.
- A running native tool is interrupted: SIGTERM goes to its process group (story "CLI: clean an explicit item list"). The sheet says so while a tool runs.
- The button reads Stop, with Esc (story "HIG pass: keyboard, colour, empty states and menus").

The Decomposition and ACs below are re-cut at the plan gate to carry these: per-item `action`, `outcome` and `bytes`, and the two counters of the preparing phase in the schema and in the `clean --items` AC.

## Decomposition

- [x] Core contract: `crates/ddc-core/src/progress.rs` with `PROGRESS_SCHEMA_VERSION` and the event types; `docs/progress.schema.json` generated from them, with a staleness test.
- [x] Core hook: `ExecEvents` in `ExecOptions`, called once before the first action, and once per item after its journal record (at once for a `not-run` item, which is never journaled).
- [x] Core: `store::last_scan_files`, the previous report's file count, read before the rescan replaces it.
- [x] CLI: `--progress json` on `scan` and `clean`; the mode (off, terminal line, JSON); the throttle; the emitter; the hook's sink.
- [x] CLI: `scan`, `clean` and `clean --items` wired; README.
- [x] DDCKit:
  - the event decoder;
  - the clean progress reducer;
  - the runner keeps event lines out of its stderr tail;
  - the popover's file count;
  - `CleanRun` passes `--progress json` and switches the phase on the `cleaning` event;
  - `native-only` items leave "frees about" and the bar's byte total (decision 2);
  - `FakeCLI` replays recorded stderr lines for `clean` (an event file played before `clean-go`, and a crash variant that plays `phase cleaning` first).
- [x] App: the sheet's two steps and the determinate bar (artboard 11).
- [x] Menu bar ADR §6 item 3: a dated note that the `cleaning` event moved into the executor (decided at this gate).
- [x] Tests; `make check` and `make app-test`; critic, then reviewer.

## Implementation Plan

Planned 2026-10-25 with the planner, after "Progress output while ddc clean runs" (DDC-001), whose counter this story emits. No spec covers it. The binding "how" is the menu bar ADR §6 item 3, as amended 2026-10-24, and the safety ADR §5: `native-only` space is not counted in the total.

### Decisions taken at this gate

1. **Where `cleaning` is written.** The amendment of 2026-10-24 left this story to decide whether the event moves into the executor. It does. A hook on the executing thread writes it after the executor's own preparation (the local-snapshot count, the keep-marker walk, the process snapshot) and immediately before the first action. It is written even when no item runs, and never when `execute` refuses (root, `review` with `--yes`).

   The sheet's step 1, "Checking again", then covers the marker walk as artboard 11 draws it. The crash rule is unchanged in direction: the event still comes before any touch. A poller cannot guarantee "on stderr before the first action", hence a hook.

   Story "Faster clean: re-check only what the selection needs" uses this event and this schema file as they are. The menu bar ADR gets a dated note when this lands.
2. **`native-only` items.** "Selected" stays what the scan measured: `Confirmation.bytes`, the class rows and the Review window's `selectedBytes` do not change, so Review and the sheet name the same number. What a clean promises leaves them out, as the terminal's "frees ~" already does (`prompt.rs`): `Confirmation.reclaimable` and `selectedReclaimable`. The sheet gains `Confirmation.counted` (the bytes without `native-only` items), which is the bar's byte total. When it differs from `bytes`, a line under the header says so: "<size> inside <runtime>'s disk image is pruned there and not counted", with the runtime named from the item's tool (Docker, Podman, Colima, …). Today the app has no such line. The bar and its "X of Y" use the events' `bytes_total`, which also leaves out items refused at the re-check; `counted` serves the header and that line before the first `items` tick.
3. **`n` counts the items the executor runs.** Items `plan_listed` refused at the re-check appear only in the final report, where the result counts them among the skipped, as it does today. So the bar can read "5 of 5" for a sheet of 7, and the result names the other two.

### The events

Each line is one JSON object on stderr with `schema_version: 1` and an `event` field.

| `event` | When | Fields |
|---|---|---|
| `phase` | Never held back, flushed. `preparing` comes after the lock, before the re-check walk; `cleaning` comes from the hook (decision 1). | `phase`: `preparing` or `cleaning` |
| `walk` | Throttled, plus each step's final tick. | `step`: `scan` (ddc scan), `recheck` (the clean's re-check: the full rescan today, the pruned walk after faster clean), or `markers`; `files`, `dirs`, `bytes`; optional `files_expected` |
| `items` | Throttled, plus a final tick. | `done` (outcomes other than `not-run`), `n`, `bytes_done`, `bytes_total` (both without `native-only` items); optional `current`: `i`, `path`, `action`, `bytes`, and `tool` while its command runs |
| `item` | One per item the executor ran, `not-run` included, after its journal record, never held back. | `i`, `n`, `path`, `action`, `bytes`, then the flattened outcome as in the clean report (`outcome`, `reason`, `name`, `commands`, `error`, …) |

**Where `files_expected` comes from:**
- `recheck`: the previous report's `totals.files`, read before the rescan replaces it;
- `markers`: the executed items' `files`;
- `scan`: absent.

**The final ticks.** The final `scan` and `recheck` ticks come from the finished report's totals, so they are exact. The live byte counter counts a hard-linked file once per link.

**The throttle.**
- `walk` and `items` ticks are at least 100 ms apart, and sent only when changed.
- The final tick of each step is exempt from the gap.
- `phase` and `item` lines are never held back.

**One emitter state, one order.**
- The emitter's state sits under its `Mutex`: the last `walk` tick sent per step and the last `items` tick. The hook's final tick and the ticker's ticks both go through it, so a tick never repeats.
- The executor calls `events.cleaning()` before it flips the counter's phase. The ticker sends `items` ticks only after the sink has written `phase cleaning`. So no `items` tick comes before `phase cleaning`, and no `walk` tick comes after it.
- Lock order: the emitter's lock first, then the counter's snapshot (its `current` lock), in the ticker and the hook alike. The executor never holds `current` while it calls the hook.

**The reducer's contract.** The phase only moves forward. A `walk` tick after `cleaning` is ignored. Counters take the larger value, and an `item` line with an `i` already seen changes nothing. An `items.done` that lags the last `item` line is normal: the counter moves before the hook writes.

### Route

1. **Core contract:** `crates/ddc-core/src/progress.rs`, following `notify.rs` and `settings.rs`.
   - `ProgressLine { schema_version, #[serde(flatten)] event }`, with `ProgressEvent` tagged by `event` (`phase`, `walk`, `items`, `item`) in the shape of `PreflightLine`.
   - The outcome reuses `exec::Outcome`, `ActionKind` and `SkipReason`. `current.tool` is an owned `String` in the event type, so it derives `Deserialize` for the round-trip of AC 1.
   - The schema describes `files_expected` as an estimate: nested items count twice, a region under-counts, and picked entries count no files.
   - `docs/progress.schema.json` is generated by `schemars` and guarded by a staleness test, as `tests/notify.rs` does.
2. **Core hook** in `exec/mod.rs`: `ExecEvents { fn cleaning(&self); fn item(&self, i, n, item, outcome) }`, held in `ExecOptions.events`, called as decision 1 says.
3. **`store.rs`:** `last_scan_files(env)` reads the version and `totals.files` only.
4. **CLI.**
   - `ProgressFormat { Json }` in `cli.rs`, as `NotifyFormat` is. `--progress` is on `ScanArgs` (in conflict with `--background`) and on `CleanArgs`.
   - In `progress.rs`, the mode is JSON when asked, else a terminal line when stderr is a terminal, else off.
   - A pure `Throttle::due(now: Instant)`, for an injected clock.
   - The emitter sits behind a `Mutex`. Each line is serialized to one buffer, written with one `write_all` and flushed, and write errors are ignored.
   - The spinner of DDC-001 becomes a ticker that serves both modes. The sink implements `ExecEvents`: `cleaning()` writes the final `markers` tick, then the phase line, then flushes.
5. **Commands.**
   - `cmd/scan.rs` emits `walk scan`.
   - `cmd/clean.rs` and `cmd/clean_items.rs` write `phase preparing` after the lock, read the expected count, emit `walk recheck` in place of the spinner, and pass `progress` and `events` to `execute`.
   - README: `### ddc scan` and `### ddc clean` name `--progress json` (the README test checks it) and link the schema.
6. **DDCKit.**
   - `ProgressEvent.swift` decodes a line, returning nil for non-JSON lines, unknown events or another version. Another version is logged once per run ("progress events of schema N ignored"), so a sheet without progress has a reason in the log. The outcome switch of `CleanReport.Item` moves into `CleanReport.Outcome.init(from:)` for reuse.
   - `CleanProgress` is the reducer. The bar fraction is by bytes, falling back to items when there are no bytes. It also tracks the in-flight stripe (none for `native-only`), the skipped count, the tool note and the step-1 counters, clamped.
   - `CLIRunner` keeps lines starting with `{` out of `stderrTail`, so logs and failures keep the `ddc:` lines.
   - `PopoverModel` passes `--progress json` to `scan`, with a `stderrLine` that yields into an `AsyncStream` for `scan.files`. The callback only decodes and yields and never waits for the main actor.
   - `CleanRun` appends `--progress json` and gains a progress event. `cleaning` switches the phase. The modification-date watcher switches it only until the first event. The crash rule becomes "saved, or the `cleaning` event seen", so "Faster clean" only deletes code.
   - `ReviewModel.Sheet` gains `progress`, and `.stopping` keeps the bar. The `native-only` totals follow decision 2.
7. **App:** `ConfirmSheet`'s cleaning view.
   - It shows "Checking again", then the item bar. The in-flight segment is striped, and static under Reduce Motion.
   - Below the bar: "i of n · X of Y · N skipped", the path under `~` or the tool's name, and "Stop interrupts <tool>" while one runs.
   - New files are picked up by the synchronized group.

## Acceptance Criteria

- [ ] **AC 1. Scan events.** `scan --json --progress json` on a temp tree:
  - every stderr line that starts with `{` decodes as `ProgressLine` and encodes back to the same value, and the other lines start with `ddc:`;
  - the staleness test ties `docs/progress.schema.json` to that type;
  - every event is `walk` with step `scan`, and the last one's `files`, `dirs` and `bytes` equal the report's `totals.files`, `totals.dirs` and `totals.scanned_bytes`.
- [ ] **AC 2. Clean events in order.** `clean --items --json --progress json` over a temp tree emits:
  - `phase preparing`;
  - `walk recheck`, the last tick with `files` equal to the saved report's `totals.files` and `files_expected` equal to the previous report's;
  - `walk markers`, the last tick with `files` equal to `files_expected`;
  - `phase cleaning`;
  - one `item` per executed item, `i` going from 1 to `n`, each with `action`, `outcome` (and `reason` when skipped) and `bytes`;
  - a final `items` tick with `done` equal to `n` and `bytes_done` equal to `bytes_total`;
  - no `items` tick before `phase cleaning`, and no `walk` tick after it.
- [ ] **AC 3. The event before the first action.** While a fake tool's command runs, stderr already holds `phase cleaning` and the journal is still empty; then SIGTERM ends the run (CLI test, stderr read on a thread).
- [ ] **AC 4. Hook order.** A core test with a recording hook and a journal sharing one log shows:
  - `cleaning` comes once, before the first journal entry;
  - there is one `item` per plan item, and the hook is also called with no items;
  - nothing is called when `execute` refuses `review` with `--yes`;
  - a walk cancelled through `cancel_walk` leaves its target kept and every item `not-run`;
  - a `native-only` item carries `bytes` on its `item` event but is not in `bytes_total`.
- [ ] **AC 5. Throttle.** A CLI unit test with an injected `Instant` feeds a changing snapshot every 10 ms for 2 s. Between 15 and 21 ticks are written, and no two ticks are less than 100 ms apart except a step's final one. `phase` and `item` lines are never held back.
- [ ] **AC 6. No JSON without the flag.** Piped `scan` and `clean --items` write no stderr line starting with `{`. The assertion of "Progress output while ddc clean runs" still holds.
- [ ] **AC 7. Popover.** `walk scan` lines from a fake runner set the popover's file count; without events the count stays empty and the elapsed time shows (DDCKit test).
- [ ] **AC 8. Sheet.** DDCKit test, with a fake CLI printing recorded event lines to stderr:
  - the step-1 counters appear;
  - `cleaning` switches the phase with no change of `last-scan.json`;
  - the sheet shows item `i` of `n` with its path, the bytes fraction and the skipped count;
  - a `native-only` item in flight adds no stripe;
  - a crash after `cleaning` with no saved file points to the journal;
  - the existing modification-date tests still pass, which covers a clean without events.
- [ ] **AC 9. `native-only` totals.** `reclaimable`, `selectedReclaimable` and `counted` leave `native-only` items out; `bytes`, `selectedBytes` and the class rows do not change; the explanatory line appears only when a `native-only` item is listed (unit test).
- [ ] **AC 10. Contracts.**
  - `ContractTests`: the schema version, the event names and the phase and step values match the decoder.
  - `CLIRunnerTests`: event lines stay out of the stderr tail.
- [ ] **AC 11.** `make check` and `make app-test` pass.

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- **Pipes.** Events are written synchronously on the executor's thread, so a stalled reader with a full 64 KiB pipe blocks `execute`. `CLIRunner` drains both pipes. Its `stderrLine` callback runs on the reader thread and must only decode and yield. Rust tests that read stderr after exit must read it on a thread once `--progress` is on.
- **Mixed lines.** The signal handler writes "stopping after the current item" raw, and `ddc:` warnings and the disclaimer share the stream. Parsers skip lines that do not decode. One `write_all` per line keeps a line from being split.
- **A closed stderr** (EPIPE) never fails a clean. The emitter ignores write errors and never uses `eprintln!`.
- **The clock.** It is an injected `Instant`, never `SystemTime`. A spurious wake-up of `park_timeout` is checked against the gap. The throttle test does not sleep.
- **Tests never run a real tool.** `Env::tool_dirs()` always adds the Homebrew prefixes, so a restricted `PATH` still finds a real `docker`. `native-only` coverage stays in the core with `FakeCommands`; real-CLI DDCKit tests only dry-run or scan.
- **Structural tests to keep green:**
  - `ContractTests.cleanHasOneOwner`;
  - the Stop shortcut check;
  - `titleCase` for any new button;
  - `structure.rs` for `cmd/background.rs`.
- **Load.** Thousands of `item` events: sheet updates are batched on the main actor, published with each tick.
- **The app ships the CLI it runs;** the schema still guards scripts.
- **The terminal clean with `--progress json`** writes `phase preparing` and `walk recheck`, then nothing while the `[y/N]` prompt waits. This is harmless.
- **Ctrl-C during the marker walk** (left by "Progress output while ddc clean runs"): the ticker, which sees the cancel flag, also cancels the shared walk counter, so one large tree stops mid-walk. A cancelled walk is treated like a failed one: `keep_markers` pushes the target as kept when `WalkOutput.cancelled` is set, so a cancel mid-walk fails closed whichever flag the executor sees first. `ExecProgress` gains `pub fn cancel_walk(&self)` for the ticker; `walk()` stays `pub(crate)`.
- **Line numbers drift.** `ReviewModel.selectedBytes` and `selectedReclaimable` are found by name, not line.

## Dependencies

- [[epics/DDC-001/stories/story-progress-output-while-ddc-clean-runs|Progress output while ddc clean runs]] (DDC-001): hard dependency, the counter
- [[epics/DDC-002/stories/story-cli-clean-an-explicit-item-list|CLI: clean an explicit item list]]
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]
- [[epics/DDC-002/stories/story-review-and-clean-window|Review and clean window]]: the clean progress state this story feeds

## Attachments

- Design canvas: https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV

---

*Last updated: 2026-10-25*

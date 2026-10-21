---
type: story
id: "story-review-and-clean-window"
epic: "DDC-002"
title: "Review and clean window"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-21
external_refs: {design_canvas: "https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV"}
tags: [app, ui, cleanup]
code_refs: []
specs: []
started_at: "2026-10-19T12:35:13.252Z"
closed_at: null
plan_updated_at: "2026-10-19T12:35:13.252Z"
---

# Review and clean window

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

This story is artboards 4 to 6 of the design canvas. The user browses what was found, by group or by item, and selects. A sheet lists every item to confirm, with live warnings. The user then watches progress and sees the result, with what was skipped grouped by cause. It is the only way the app cleans (ADR "Menu bar app: Swift shell over the ddc CLI", §7).

## Decomposition

- [ ] Window: a sidebar of classes and groups, and a table of items with:
  - the path under `~` (or the volume), the size and the frees estimate;
  - the project's idle days and whether it has a lock file;
  - notes (`no-lockfile`, `app-owned`, `kept-by-marker`, `excluded`) and the `not_selected` reason
- [ ] Groups / items toggle; search; sort by size, path and idle days; Scan again in the header (artboard 4)
- [ ] Selection: defaults from the report; `review` items start unselected; `user-data` and `ignored` items cannot be selected, nor items ddc has no command for ("Manual only": `trash`, `vm.colima`, `vm.lima`, `vm.orbstack`); an `incomplete` finding says that part of it could not be read
- [ ] Reveal in Finder; "Protect this folder" writes `.ddc-keep` after a confirmation
- [ ] Confirmation sheet:
  - counts per class;
  - every item with its action (delete, Trash, the tool's command);
  - for each `review` item the Trash would take, Delete permanently, defaulting to the app preference "Delete review items instead of moving them to the Trash" (off: the Trash); a `review` item cleaned by its tool has no such switch;
  - warnings for the Trash and for local snapshots;
  - Cancel and Clean
- [ ] Live warnings: the sheet opens at once and runs `clean --items --dry-run --json` over the listed items (no rescan); the predicted skips stream in per item ("Spotify is running · will be skipped"); Check again reruns it
- [ ] Clean: the sheet's Clean action, and nothing else in the app, runs `clean --items <file> --json` with exactly the confirmed items (each with its class, action and `permanent`). It shows two phases:
  - the preparing phase (the CLI's rescan), with the elapsed time;
  - the cleaning phase, which starts when the CLI saves its rescan to `last-scan.json`.

  Item `i` of `n`, the current path and the bytes processed arrive with "CLI: progress events", which feeds the same progress state and adds `--progress json`. Cancel stops after the current item; a clean still waiting behind another run never starts. While the clean and the popover's rescan after it run, the sheet sets the `busy` input of the icon state of "Menu bar popover", so the arc turns as on artboard 3.
- [ ] Result: deleted, moved to the Trash, tools run, skipped and failed; free space before and after for each volume; skipped items grouped by cause (count and size per cause, the items inside), in plain language; Open journal

## Implementation Plan

Plan from the `projectstore:planner` pass (2026-10-19). No spec covers the story. The design is the ADR "Menu bar app: Swift shell over the ddc CLI" (§1, §2, §6 items 1, 2, 4 and 5, §7) plus artboards 4 to 6. It lands as "Menu bar popover" did:
- the contracts and the logic go in DDCKit (Foundation and Observation), tested by `swift test`;
- the app target gets a thin `NSWindow` controller and SwiftUI views;
- no Rust change.

### Decisions taken at this gate

1. **Progress behind a seam.** The window does not wait for "CLI: progress events". The app never passes `--progress json` until then: `CleanArgs` has no such flag, and clap refuses an unknown flag with exit 2 on every clean. Progress is display only; consent, the per-item checks, cancel and the result all exist today.
   - **The signal.** The switch from preparing to cleaning is the first change of `last-scan.json`'s modification date after the clean process starts. While the clean holds the exclusive lock nothing else writes that file (`crates/ddc-cli/src/cmd/clean_items.rs`). The baseline is taken when the process starts, not when the clean is asked for, because a scan queued ahead of it writes the file too.
   - **How the app sees it.** It polls the file's date every 200 ms while preparing. A watcher on the file would die with the CLI's atomic rename, so it would have to watch the folder.
   - **Why not a stderr line.** An additive `ddc: scan saved` would need a Rust change. "CLI: progress events" brings the real events, and the poll goes then.
   - **The same date says whether anything was touched.** The CLI saves the rescan before its first item, so an unchanged file means nothing was.
2. **Sidebar keyed by (class, group).** It has the class sections of artboard 4 in class order, each listing the groups of `rules --json` that have findings of that class, plus a "User data · report only" section.
   - **Groups, not the canvas's categories.** The canvas's categories ("Package caches", "Idle projects") are in no contract. The groups come from the CLI, so a new rule lands in the right group without an app release.
   - **Group requests.** A group request from the popover selects the first (class, group) entry in class order that holds a default-selected item of that group, else the group's first entry.
   - **Without `rules --json`.** Every finding is in "Other", and a `native-only` finding cannot be selected, since `switchable` is unknown.
3. **Groups view** rows are rules (one checkbox per rule, mixed when part of its items is selected). **Items view** ("Files" on the canvas) lists every finding biggest first. Ignored findings appear only in the Items view, dimmed, with a disabled checkbox and the reason ("type switched off", "under the minimum size"); user-data findings appear only under "User data".
4. **The tool's command** has no field before the run (the dry run's `would-run` carries no command). The sheet says "cleaned by its own tool"; the result names the commands the report's `native` outcome ran. An additive field in `rules --json` is a follow-up if the canvas wording is wanted.
5. **Progress and result live in the sheet** (artboard 5's cleaning phase, then artboard 6). Closing the window does not stop the clean; the sheet comes back with its state when the window reopens.
6. **The payload keeps the items the dry run predicts to skip.** They were confirmed, and the clean checks them again. When every item is predicted to skip, the sheet says that nothing can be cleaned right now.
7. **After Done.** An item the report lists as deleted or moved to the Trash is locked ("Cleaned just now") in any report scanned before the clean ended. That covers the clean's own rescan and the report the popover held, so a removed item never comes back selected. While the popover's rescan after the clean runs, the window says it is scanning again and Clean is off; it reloads when the scan lands. A finding the tool cleaned (`native`) keeps its folder and stays.
8. **A clean that ends without a report.** The `last-scan.json` date decides.
   - **Unchanged.** The CLI never reached its first item: "Nothing was touched", and no rescan. This covers exit 75, exit 2, and a cancel or a crash while preparing.
   - **Changed.** Items may be gone (`.crashed`, exit 1 from a journal write failure mid-run, or a cancel after the save but before the handler): "It stopped before it could report; the journal lists what was done", with Show the journal on the CLI's `actions.jsonl` (`<data>/logs` with `DDC_DATA_DIR`, else `~/Library/Logs/DeepDriveCleaner`), and a rescan.
   - **The rule.** The popover rescans after a clean only when the CLI saved its rescan or printed a report.
9. **Settings changed while the window is open.** The window reads `settings.json` itself.
   - **When the rows are rebuilt.** With the current settings, on every request from the popover, after Done, whenever the file's date changed when the window becomes key, and just before the sheet opens. A type switched off in Settings therefore leaves the selection at once, and the sheet captures under the current settings.
   - **The open sheet.** It shows the items it captured. At Clean, the payload is checked once more against the current settings, and an item a settings change made ignored since then is left out, so the app never lists an ignored item (ADR §6 item 1).
   - **The CLI.** It applies the current roots and exclusions itself.

### After the review (2026-10-19)

The `projectstore:reviewer` pass found no blocker. These came out of it:

- **Quit waits for the clean's CLI process only, never for the rescan after it.** A clean still waiting behind another run does not hold Quit; it never starts once the app is gone.
- **Cancel on a clean still waiting closes it at once** ("Cancelled before it started"), instead of after the run ahead ends.
- **A report on stdout is read whatever the exit**, a crash after the handler included. Decision 8 applies only when there is nothing to read.
- **The sheet's Clean is not the default button**, so Return never starts a clean. The window's "Clean N items…" stays the default; it only opens the sheet.
- **The `--items` file has its keys sorted**, so one list always gives the same bytes.
- **Deleting a `.ddc-keep` the window wrote unlocks its row** at the next rebuild.
- **The replayed `clean-report.json`** was recorded with delete items only, so the recording never touched a real Trash. The deleted `py1/.venv` and the trashed `.ollama/models` were then added by hand, so it answers the whole list the test sends.
- **Open for the manual check.** AppKit may not close a window while its sheet shows. The cleaning text therefore says the user can keep working, and makes no claim about closing the window.
- **Stop, and Esc in the filter (2026-10-21).** "HIG pass: keyboard, colour, empty states and menus" (DDC-002) changes some of the names and keys used here:
  - **Stop.** The cleaning sheet's Cancel becomes Stop and gets Esc. The ACs here that say Cancel mean that button.
  - **Title case.** Buttons and menu items move to title-style capitalization: "Reveal in Finder" is now "Show in Finder", "Check again" is "Check Again", and "Protect this folder" is "Protect This Folder…". The ACs here mean those.
  - **Esc.** It still closes the window as decided below. The exception is a filter field holding text: there Esc clears it first.
  - **Return.** While the filter holds text, Return stays in the field.
- **Close returns to the popover (owner, 2026-10-19).** The window's Close button, and Escape, its shortcut, close the window and open the popover under the icon, so the user lands where they came from. The title bar's close box only closes the window, as any window's does: the Close button is the window's way back, the close box is window chrome. ⌘W does nothing yet: the app has no main menu, so ⌘W, and copy and paste in the search field, wait for one (a follow-up). An already open popover stays as it is.
- **Scans paused (owner, 2026-10-19; "Menu bar popover").** While paused, Scan again (and Scan now with no report yet) gives way to "Scans paused". The rescan after a clean still runs: it is part of the clean.
- **Follow-up.** A finding that is itself a scan root counts in the popover's "Selected to clean", but the window and `build_plan` leave it out. It is an edge case; "Menu bar popover" can drop it from its totals.

### Placement, in order

1. **Contracts.** Pure; the tests come first.
   - `ScanReport.swift`: add `ruleClass`, `removal`, `dev`, `ino`, `notes`, `hint`, `incomplete` and `Project.lastActivity` to `Finding`. `RemovalKind` and `Note` decode strictly, like `NotSelected`.
   - New `ItemList.swift`: the `--items` contract, `schemaVersion` 1, pinned to `LISTED_SCHEMA_VERSION`. `ListedAction.derive` mirrors `plan::action_for` over the report's effective `removal`:
     - user data (class or rule class) and `none` → nil;
     - `native-only` → only when the rule is `switchable`;
     - `trash` → `delete` when a review item is permanent.
   - New `CleanReport.swift`: the clean report and the dry-run lines (`PreflightLine`, `Prediction`, all 18 `SkipReason` values, `Outcome`, `VolumeSpace`), with the version read first.
   - New `CleanWording.swift`: the one table of plain language for skip reasons, causes, notes, `not_selected` reasons and actions.
2. **Selection.**
   - `ReportSummary.swift`: extract the per-finding decision into one function that the popover and the window share, so "Selected to clean" equals the window's starting selection.
   - New `ReviewItems.swift`: rows built once per snapshot of report, settings and rules. It holds:
     - selectability: not user data, not ignored under the current settings, not a scan root, and an action exists. The app is the only gate for ignored items: the CLI never drops a listed item for its type;
     - the default selection, restricted to the request's classes;
     - the sidebar, search over the path (also as written under `~`) or the rule id, and sort.
   - New `Confirmation.swift`: the value captured when the sheet opens. It holds the items in sheet order, counts and bytes per class, and `permanent` per review item, starting from the preference. `itemList()` checks selectability again, so a forged item never reaches the payload.
   - New `KeepMarker.swift`. It opens the folder with `O_DIRECTORY|O_NOFOLLOW`, checks `(dev, ino)` with `fstat`, then creates `.ddc-keep` through `openat` with `O_CREAT|O_EXCL|O_NOFOLLOW`. That is the race-free form the CLI's guard uses. A marker already there counts as done; a read-only folder (Go's module cache) gets its own message. The marker's name is pinned to `KEEP_MARKER` (`crates/ddc-core/src/rules/visitor.rs`) by a drift test.
3. **Runner.** In `CLIRunner.swift`:
   - a stdout line callback that still returns all of stdout;
   - a per-run ticket with `cancel(ticket)`: SIGTERM when that run is running, otherwise the run is dropped and never started. Today `cancel()` kills whatever runs, the popover's scan included, and a queued clean still starts;
   - an `onStart` callback.

   The current `run(_:_:stderrLine:)` stays as a convenience, so `PopoverModel` and the fakes barely change.
4. **Clean run.** New `CleanRun.swift`, the only owner of the clean and dry-run kinds.
   - The dry run streams predictions by path.
   - `cleanConfirmedItems()` is the single entry point. It:
     - writes the list to a unique temp file (0600), kept until the run ends: the runner's stdin is `/dev/null`, so `--items -` would read an empty list;
     - runs `clean --items <file> --json`;
     - tracks waiting → preparing → cleaning → stopping;
     - decodes the report off the main actor;
     - asks the popover for one rescan when the CLI saved its rescan or printed a report (decision 8).
   - `CleanRun.swift` is an extension of `ReviewModel`: the clean's `Task` is created there, in `cleanConfirmedItems()`, and kept on the model, so closing the window never loses it. The sheet only calls that method.
   - Busy (75), refused (2), error (1) and `.cancelled` with or without stdout each map to the Technical Notes' reading.
   - New `CleanResult.swift`: counts and bytes per outcome, the volumes before → after (the total from the snapshot's `volumes`), and the skips grouped by cause.
   - `PopoverModel` and `IconState` take a clean in flight as `busy` (the icon's VoiceOver text says "cleaning"). `PopoverModel.refresh()` skips reloading `last-scan.json` while a clean runs: the CLI's rescan is the state before the clean.
   - `DataDirectory` learns the CLI's log folder, for Show the journal.
5. **Model.** New `ReviewModel.swift`, `@MainActor @Observable`, which conforms to `ReviewRouting`.
   - `open(_:)` applies the request and calls the injected `present`. When `confirm` is set and the selection is not empty, it opens the sheet, which starts the dry run.
   - While a sheet, a clean or a result is showing, a request only brings the window forward.
   - Its dependencies are closures (snapshot, `setBusy`, rescan, log, clock), as `PopoverModel.init` does. That avoids a circular init.
   - `AppPreferences` gains `DeleteReviewItemsPermanently` (false), read only; Settings adds the switch.
6. **App target.**
   - `ReviewWindowController.swift`: a lazy `NSWindow` like `SettingsWindowController`. It is the `present` closure.
   - `ReviewView.swift` (artboard 4): the sidebar, the Groups/Items toggle, search, sort, the rows, Reveal in Finder, Protect…, and a footer with the selection and "Clean N items…".
   - `ConfirmSheet.swift` (artboard 5 and its cleaning phase): the only caller of `cleanConfirmedItems()`.
   - `CleanResultView.swift` (artboard 6). Show the journal reveals `report.journal`.
   - `AppDelegate.swift`:
     - replaces `ReviewRouterStub` with the review model, built before the popover model;
     - on Quit during a clean, cancels it and returns `.terminateLater`, then replies when the CLI exits: a child process outlives the app.
7. **Checks.**
   - `make app-test`, and the unsigned `xcodebuild … CODE_SIGNING_ALLOWED=NO`.
   - Then `projectstore:reviewer`.
   - The owner runs `make app` and the manual checks.

### Pitfalls to hold

- The list contract is strict (`deny_unknown_fields`, no empty list, no duplicate path; `permanent` is required on a review item listed as `delete` and allowed nowhere else). Each breach is exit 2, so Clean is disabled on an empty selection.
- Paths are sent exactly as decoded: no standardising, no resolving, `.withoutEscapingSlashes`. Rows are keyed by the finding's index, never by a `String` set: Swift `String` equality would merge the NFC and NFD forms of one name.
- The window's snapshot never swaps on its own, and the popover does not reload `last-scan.json` during a clean. After Done the window reloads and keeps the selection by `(path, dev, ino)`, the path compared as UTF-8 bytes (decision 7 for what was removed).
- The clean's `Task` lives in the model, never in a SwiftUI `.task`. Pressing Clean twice starts one clean.
- Tests never touch the real Trash, the real data directory or TCC.db. A test with the real CLI runs a dry run only, over a temporary home.
- The structural test strips comment lines first. Enum cases are named `.cleaning`, so the `\.clean\b` check stays exact.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Unit test: the `--items` payload is exactly the confirmed list, each item with its class, action and `permanent`; no item the user did not see in the sheet is sent
- [ ] Unit test: a `review` item's `permanent` starts from the app preference and changing one item changes only that item; a `review` item cleaned by its tool has no switch and is listed with `permanent` false
- [ ] Structural test, comment lines stripped:
  - the `.clean` and `.dryRun` run kinds appear in no source but `CLIRunner.swift` and `CleanRun.swift`;
  - `"--items"` appears only in `CleanRun.swift`;
  - `cleanConfirmedItems(` appears only in its definition in `CleanRun.swift` and once in `ConfirmSheet.swift`;
  - no other app or DDCKit source calls it, so notifications, the scheduler and the popover reach only the window or the sheet
- [ ] Unit test: `user-data` and `ignored` findings cannot be selected, and `review` findings start unselected
- [ ] Unit test: the popover's review request ("Menu bar popover": a group, the classes switched on, and whether Clean selected… asked for the sheet) seeds the window's group filter and selection, opens the confirmation sheet when asked, and the router reports the window opened so the attention badge clears; opened from Clean selected…, the selection is exactly the popover's "Selected to clean" (same items, same bytes), and the sheet opens without starting a clean
- [ ] Test with a fake CLI (a script that replays recorded dry-run and result JSON, and replaces `last-scan.json` part-way, as the CLI saves its rescan):
  - the sheet shows the predicted skips;
  - then the preparing phase, then the cleaning phase within 1 s of the replace;
  - then the result with skips grouped by cause;
  - after Done, no item the report lists as deleted or moved to the Trash can be selected;
  - every skip reason the executor has (`docs/clean.schema.json`) maps to text
- [ ] Test with a fake CLI that writes its dry-run lines with a delay: the sheet is shown before the first line arrives, and each warning appears on its item as its line arrives
- [ ] Unit test: Cancel stops only the clean:
  - a clean still waiting behind a scan never starts, and the scan goes on;
  - a running clean gets SIGTERM;
  - on `.cancelled` the result is read from stdout when it is not empty.

  Without a report:
  - a clean that never saved its rescan says nothing was touched, and nothing rescans;
  - one that did (the fake's crash after the save) says the journal lists what was done, and the popover rescans once.

  The icon stays busy until that scan ends.
- [ ] Unit test: Protect this folder writes an empty `.ddc-keep` into the item's folder only after the confirmation, refuses a folder whose `(dev, ino)` changed or that is now a symlink, and the item can no longer be selected
- [ ] Test with the real CLI over a temporary home, skipped when `target/debug/deepdrivecleaner` is missing:
  - a wrapper script sets `HOME` and `DDC_DATA_DIR` for the child;
  - the home holds findings of each action and class: `delete` safe and rebuildable, `native` safe, `native-only` review, `trash` review, and a `review` item deleted permanently;
  - the dry run accepts the list the window builds for every selectable finding: exit 0, no `changed-since-scan`, no `action-not-allowed`
- [ ] Manual check on the reference machine:
  - cleaning a few selected `safe` items from the window frees space, the before and after values are shown, and the journal lists exactly those items;
  - search, sort, the Groups/Items toggle, Check again and Reveal in Finder work
- [ ] Manual check: Cancel during a clean stops after the current item, and the result lists what was done
- [ ] Unit test: while scans are paused Scan again starts nothing, and a clean still asks for its rescan
- [ ] Manual check: the Close button returns to the popover, which stays open after the window is gone

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- From "CLI: clean an explicit item list":
  - `clean --items --json` exits 0 whenever it produced a report, failed items included. After a cancel it ends by the signal (`.cancelled`) with the report on stdout. The app decodes stdout in both cases.
  - The CLI installs its handler only after the rescan is saved. A cancel before that ends it by the signal with empty stdout. A cancel any later gives a report, possibly with every item `not-run`. So on `.cancelled` the app decodes stdout when it is not empty; otherwise decision 8 decides between "nothing was touched" and the journal message.
  - The CLI saves its rescan before executing, so the popover rescans after every clean that got that far, cancelled ones included (decision 8: not after one that never saved it).
  - What was left undone is read from the report, never from how the process ended: `cancelled`, the `not-run` items, and an interrupted tool, which shows as a `native` outcome whose command has `success: false`. A signal that arrives after the last item started ends the CLI by the signal with a complete report and `cancelled: false`.
  - The dry run streams one JSON line per item on stdout. `CLIRunner` (`app/Packages/DDCKit/Sources/DDCKit/CLIRunner.swift`) returns stdout only at EOF and streams only stderr lines, so showing each warning as its line arrives needs a stdout line callback.
  - The dry run takes the shared lock and exits 75 (`.busy`) while a terminal `ddc clean` holds it, so the sheet needs a busy state. Exit 2 (`.refused`) on a list the app built is an app bug.
- From "CLI: shared settings, types and the notify decision": the app passes no positional roots and no `--exclude` to `scan`, `clean --items` or its dry run; all three resolve them from `settings.json`. A settings change between review and clean can turn items into `changed-since-scan` or `kept`; the result screen shows them. `--items` with an `--exclude` that drops a protected folder exits 2. `ignored` (absent when false), `volumes` (absent when empty) and `Totals.ignored_count` / `ignored_bytes` (absent when 0) are optional in the scan report, so `ScanReport.swift` decodes them with `decodeIfPresent`; a strict decode fails on the first report without them.
- Skip reasons come from the executor (`SkipReason`), so their wording lives in one table in the app with a test that covers every variant.
- The rescan before cleaning (about 40 s on the reference machine) shows as the first phase of the progress.
- A dry run predicts; the clean re-checks everything again, so a prediction never replaces a check.
- Until "CLI: progress events", the app never passes `--progress json`: clap refuses an unknown flag with exit 2.
- The report's `removal` is the effective method (a review finding's `delete` is rewritten to `trash`, `crates/ddc-core/src/report/mod.rs`); the action is derived from it, never from the `removal` of `rules --json`, or the CLI answers `changed-since-scan`.

## Dependencies

- [[epics/DDC-002/stories/story-app-skeleton-and-build-pipeline|App skeleton and build pipeline]]
- [[epics/DDC-002/stories/story-cli-clean-an-explicit-item-list|CLI: clean an explicit item list]]
- [[epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision|CLI: shared settings, types and the notify decision]]
- [[epics/DDC-002/stories/story-menu-bar-popover|Menu bar popover]]
- Fed later by [[epics/DDC-002/stories/story-cli-progress-events|CLI: progress events]] (item progress; not a prerequisite)

## Attachments

- Design canvas: https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV

---

*Last updated: 2026-10-21*

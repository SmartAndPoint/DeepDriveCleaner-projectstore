---
type: story
id: "story-whole-disk-in-the-app"
epic: "DDC-002"
title: "Whole disk in the app"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-30
external_refs: {design_canvas: "https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV"}
tags: [app, ui, whole-disk]
code_refs: ["app/DeepDriveCleaner/DiskView.swift", "app/DeepDriveCleaner/DiskWindowController.swift", "app/DeepDriveCleaner/AppDelegate.swift", "app/DeepDriveCleaner/PopoverView.swift", "app/DeepDriveCleaner/ReviewView.swift", "app/DeepDriveCleaner/ReviewWindowController.swift", "app/DeepDriveCleaner/StatusItemController.swift", "app/Packages/DDCKit/Sources/DDCKit/DiskModel.swift", "app/Packages/DDCKit/Sources/DDCKit/DiskReport.swift", "app/Packages/DDCKit/Sources/DDCKit/DiskSummary.swift", "app/Packages/DDCKit/Sources/DDCKit/CLIRunner.swift", "app/Packages/DDCKit/Sources/DDCKit/DataDirectory.swift", "app/Packages/DDCKit/Sources/DDCKit/ReviewModel.swift", "app/Packages/DDCKit/Tests/DDCKitTests/DiskModelTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/DiskReportTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/DiskSummaryTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/CLIRunnerTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ContractTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/FakeCLI.swift", "app/Packages/DDCKit/Tests/DDCKitTests/PopoverSupport.swift", "app/Packages/DDCKit/Tests/DDCKitTests/Fixtures/disk-report.json", "app/Packages/DDCKit/Tests/DDCKitTests/Fixtures/disk-cancelled.json", "crates/ddc-core/tests/disk.rs", "README.md"]
specs: []
started_at: "2026-10-28T10:45:53.178Z"
closed_at: "2026-10-30T06:45:44.524Z"
plan_updated_at: "2026-10-28T10:45:53.178Z"
---

# Whole disk in the app

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

This story shows the report of "Whole disk: what lies outside the home folder" (DDC-002) in the app. It is read-only: no button here frees anything. The actions belong to "Whole disk: free space with the owning tool".

**Where it opens:**
- in the popover, a "Where does the rest go?" link under the totals;
- in the Review window, a "Whole disk" row at the bottom of the sidebar.

**What it shows:**
- **A bar of the startup container.** The home folder is drawn with what the last scan can free in the class colours, and its remainder in a neutral tone. The other areas and "not counted" are drawn in neutral tones too.
- **One row per area**, with:
  - its size, or "at least" for a partial area;
  - what the area is and what frees it;
  - for the home folder, the source and the date: "from the scan at 10:00", or "measured now".
- **"Not counted" is a row of its own**, with its explanation and the number of unreadable folders. An inconsistent report says so in the row and does not draw a negative segment.
- **The date of the whole-disk scan** and a "Scan whole disk" button, with progress and Stop. A stopped scan shows its partial rows as partial.

**How it runs:**
- `ddc disk` is read-only and takes no ddc lock. The app runs it outside the `CLIRunner` queue, in a lane of its own, so a scan of several minutes never delays Scan now, the scheduled scan or a clean.
- Only one whole-disk scan runs at a time.
- The app never starts it on its own. The scheduled scan stays the home folder's.

## Decomposition

Rough until the plan gate.

- [x] **DDCKit:**
  - `DiskReport` decodes `last-disk.json` (schema version 1). A report of another version is not decoded loosely; the view shows that the app and the CLI do not match, as for the other reports (menu bar ADR §1).
  - A `DiskModel` holds the running state, Stop and the progress.
  - The `CLIRunner` gains a separate lane for `.disk` that does not wait on, or block, the queue.
- [x] **App:**
  - the Whole disk view, used both in a window and as the Review window's detail;
  - the popover link;
  - the sidebar row. In the Review window the detail hides the picker, the filter and the Clean footer, and keeps Close with Esc (reviewer, 2026-10-28).
- [x] **Amendment to the menu bar ADR, §1**, before code (written 2026-10-28): every run but `disk` goes through one queue. `ddc disk` takes no lock and writes nothing another command reads, so it runs beside the queue in a lane of its own, one at a time. `ddc disk free` takes the lock and stays in the queue, and in the app the two never run at once: each button is disabled while the other's run is queued or running ("Whole disk: free space with the owning tool"). Reworded after the critic passes of 2026-10-28.
- [x] **Design:** artboard 13, "Whole disk (2026-10-28)", on the design canvas, with a note on the popover link. Its home-folder split by class is sample data; the app colours the classes the popover has switched on.

## Implementation Plan

Plan from the `projectstore:planner` pass (2026-10-28). No spec covers the story. The design is §1 of the ADR "Menu bar app: Swift shell over the ddc CLI", with its amendment of 2026-10-28, and artboard 13 of the design canvas. It lands as the earlier app stories did: logic and tests in DDCKit, views in `app/DeepDriveCleaner/`.

Code for steps 1-3 was started before the critic's first pass. That pass (REVISE, 2026-10-28) changed the home split and the runner's map, the second (REVISE) made the split follow the popover's switched-on classes, and the third approved; the plan below is the approved one. Steps 4-6 were written after the approval.

1. **The report.**
   - `DataDirectory.lastDisk` names `last-disk.json`.
   - `DiskReport.swift` mirrors `ScanReport`: it reads the version header first, and a report of another `schema_version` raises `otherSchema`. The view then shows "written by another version of ddc (schema N)", as the popover does.
   - The six enums are strict `String` enums; an unknown value is an error.
   - `ContractTests` check that `DiskReport.schemaVersion` equals `DISK_SCHEMA_VERSION` and that each enum's cases equal the values in `docs/disk.schema.json`.
2. **The runner.**
   - Today `CLIRunner` keeps one `running` slot and one `cancelRequested` flag. Two runs at once would overwrite them, and Stop of a clean would miss its process. They become a map keyed by an internal run id (a run need not carry a ticket), each entry with its process, its ticket and its own flag, before the lane is added.
   - `cancel(ticket)` first looks for a running entry with that ticket and sends it SIGTERM. Only when none runs is the ticket kept, so a run still waiting never starts.
   - `Kind.disk` runs at once in its own lane and never reads or sets `last`. A second `.disk` request joins the running one.
   - The type's doc is updated: every run but `disk` is serialised; `disk` takes no lock and writes nothing another command reads. `Options.onStart` says "after every earlier run of the queue", which the disk lane does not wait for.
3. **The summary.** `DiskSummary` is pure, so the bar and the rows are tested together.
   - An area that is `partial` reads "at least".
   - "Not counted" is a row of its own, with the unreadable count. Its bytes are negative exactly when the report is `inconsistent`; then it draws no segment and the row says why, from the report's causes.
   - The home row says "from the scan at 10:00" or "measured now".
   - The home segment is split into the class colours only when the home row comes from `last-scan` and its `as_of` equals the loaded scan report's `generated_at`. Otherwise the segment is neutral.
   - The split uses the popover's numbers: `ReportSummary` with `settings.json` applied, each class's `selectedReclaimable` (what cleaning the default selection frees), never the bytes the classes found, and only the classes the popover has switched on (`PopoverModel.included`). So the coloured part's sum is the home row's "can be freed" and the popover's number, `reclaimable(including: included)`, and it follows the popover's toggles.
   - After a clean that keeps the report's `generated_at` (`clean --items` edits the saved report, ADR §6.1), the split still applies: the coloured part is the post-clean number, while the home row's size stays the one measured before the clean until the next whole-disk scan. The coloured part is clamped to the home row's size.
4. **The model.** `DiskModel` is `@MainActor @Observable`, built like the scan path of `PopoverModel`.
   - It loads `last-disk.json` when the view appears and after each run ends, and decodes it again only when the file's modification date changed.
   - `scanDisk()` runs `disk --json --progress json` and joins a run already going. The app passes no `--data-dir`; the CLI inherits `DDC_DATA_DIR`, as for the scan.
   - Progress comes from `walk` events with step `disk`. `disk` sends no `files_expected`, so the bar is indeterminate, with file and byte counters.
   - `stop()` cancels the run's ticket.
   - The outcome is read in this order. `.cancelled` comes first: the CLI prints the partial report on stdout, and the model shows it with "stopped". When stdout is empty or does not decode, the previous report stays on screen with "stopped". Any other failure shows a message worded for this view (`CLIFailure.message` speaks of the scan) and keeps the previous report, unless `last-disk.json` changed, in which case the file is read again, as the popover does.
   - On quit, the app leaves a running whole-disk scan alone. SIGTERM would replace the last full report with a partial one. The orphaned CLI finishes: it saves `last-disk.json` before it prints, a print to the closed pipe fails with BrokenPipe, which it handles, and its progress lines to stderr are ignored when they fail.
5. **The views.**
   - `DiskView` is used both as the Review window's detail and in its own window, `DiskWindowController`.
   - The Review sidebar gets a "Whole disk" row at the bottom when the sidebar shows. While that detail is open, the header's Groups/Items picker, the filter and the Clean footer are hidden, because nothing in this view frees space.
   - The popover gets a "Where Does the Rest Go?" link under the totals. It is title case, because `ContractTests.titleCase` requires it.
   - `AppDelegate` builds one `DiskModel`, so the window and the Review detail share one run. `ScheduledWork` never gets it.
6. **Tests.**
   - `DiskReportTests`: fixtures for partial, inconsistent, home from `last-scan`, home from `walk`, no runtimes, and cancelled. These fixtures are written by hand against the schema, never from a real `ddc disk`.
   - `CLIRunnerTests`:
     - a `disk` run held at a gate does not delay a scan or a clean;
     - two `.disk` requests start one process;
     - with a disk run and a scan running at once, cancelling either ticket stops that run only, in both directions.
   - `DiskSummaryTests`: the split condition (it splits only for the same scan, from `selectedReclaimable`), the coloured sum equal to the home row's number and to the popover's for each set of switched-on classes, an inconsistent report drawing no "not counted" segment, partial rows, and the simulator row for each `simctl` status.
   - `DiskModelTests`: the join, Stop showing the partial rows, a cancel with empty stdout keeping the previous report, progress, and the reload when the file's date changes, on appear and after a run.
   - FakeCLI gains a `disk` branch, and `FakeRunner` gets an explicit `.disk` case.
   - Contract tests: `run(.disk` appears only in `DiskModel.swift`, `scanDisk(` only in the model and the view, and `ScheduledWork.swift`, `PopoverModel.swift` and `AppDelegate.swift` contain neither. The patterns are `run\(\s*\.disk\b` and `"disk"`, because `WalkStep.disk` already exists.
   - Quit: a contract test checks that `AppDelegate`'s termination path does not cancel the disk model's run.

## Acceptance Criteria

- [x] Test: `DiskReport` decodes a fixture of every report shape: partial, inconsistent, home from `last-scan`, home from `walk`, and no runtimes. A report of another schema version is not decoded and shows the mismatch. Met by `DiskReportTests` and the disk contract test, which already pass. — evidence: `DiskReportTests`, `DiskModelTests.loadStates`, `ContractTests.diskContracts`, `the_apps_disk_fixtures_are_reports_the_cli_could_write` (crates/ddc-core/tests/disk.rs)
- [x] Test: the home segment is split by class only when the disk report's home row is the loaded scan, and only for the classes the popover has switched on. The coloured part's sum equals the home row's "can be freed" and the popover's number, `reclaimable(including:)` over those classes, with one class or none switched off. — evidence: `DiskSummaryTests.bar`, `.splitFollowsThePopover` ([.safe], [.rebuildable], []), `.splitOnlyForTheSameScan`
- [x] Test: an inconsistent report (negative "not counted") draws no segment for it and its row names the causes. — evidence: `DiskSummaryTests.stopped`
- [x] Test with a fake CLI: a `disk` run held at a gate does not delay a scan or a clean queued in the `CLIRunner`, and a second "Scan whole disk" joins the running one. — evidence: `CLIRunnerTests.diskBesideTheQueue`, `.diskDoesNotWait`, `.joinedDisk`; `DiskModelTests.join`
- [x] Test: Stop sends SIGTERM, and the model then shows the cancelled report's partial rows; a cancel with nothing on stdout keeps the previous report. — evidence: `DiskModelTests.stopShowsPartial`, `.stopWithoutReport`
- [x] Test: with a disk run and a scan running at once, cancelling one ticket stops only that run, in both directions. — evidence: `CLIRunnerTests.ticketsWithTwoRuns(diskFirst:)` both ways
- [x] Test: the model reads `last-disk.json` again when its modification date changes, on appear and after a run. — evidence: `DiskModelTests.reloadOnDate` (on appear), `.failureReloadsAChangedFile` (after a run)
- [x] Test: quitting the app does not cancel a running whole-disk scan. — evidence: `ContractTests.diskCallers` (no `disk?.stop(` in AppDelegate; the terminate methods do not mention disk)
- [x] The §1 amendment is accepted after a critic pass. — evidence: critic APPROVE on the third pass, 2026-10-28; the ADR frontmatter `review_status: reviewed`
- [x] Test: nothing in the scheduled path runs `disk`. A contract test names the view's button as the one caller. — evidence: `ContractTests.diskCallers`
- Manual checks (2) moved to [Owner checks: manual acceptance across stories](../../../ops/owner-checks-manual-acceptance-across-stories.md) on 2026-10-30, by the owner's decision.

## Final Summary

**What changed** (code commit 50d3e08, released in 0.4.0 with c164663):
- A Whole disk view, opened from the popover and the Review window: a bar of the container and one row per area from `last-disk.json`. The home segment is split by class only when it comes from the loaded scan.
- `ddc disk` runs in its own lane beside the `CLIRunner` queue. Runs are keyed by ticket, so Stop ends only its own run, and a second request joins the running one.
- On demand only: nothing in the scheduled path runs `disk`, and quitting the app does not cancel a running whole-disk scan.
- The §1 amendment of "Menu bar app: Swift shell over the ddc CLI", accepted after a critic pass.

**Tests run:** `DiskReportTests`, `DiskModelTests`, `DiskSummaryTests`, `CLIRunnerTests` (the disk lane and tickets), `ContractTests.diskContracts` and `ContractTests.diskCallers`, and `the_apps_disk_fixtures_are_reports_the_cli_could_write` in `crates/ddc-core/tests/disk.rs`.

**Remaining risk:** two manual checks moved to the owner list: the view in light and dark with its bar matching the rows, and that macOS does not stop the orphaned CLI when the app quits (the contract test shows only that the app does not cancel it).

## Dependencies

- "Whole disk: what lies outside the home folder" (DDC-002): the report this view shows
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]:
  - §1, one runner;
  - this story amends §1 for a read-only lane beside the queue.

## Attachments

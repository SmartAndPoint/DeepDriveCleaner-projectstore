---
type: story
id: "story-menu-bar-popover"
epic: "DDC-002"
title: "Menu bar popover"
status: in-progress
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-19
external_refs: {design_canvas: "https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3"}
tags: [app, ui]
code_refs: []
specs: []
started_at: "2026-10-19T07:50:47.300Z"
closed_at: null
plan_updated_at: "2026-10-19T07:50:47.300Z"
---

# Menu bar popover

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

This story is what one click on the status item shows (artboard 1 of the design canvas), and the status item's icon states (artboard 3):
- how much is selected to clean, and how much was found by class;
- the biggest finds;
- Scan now;
- the Full Disk Access status and the next scheduled scan.

It also covers the states with no report yet, a scan running, nothing worth cleaning, and a report that cannot be read.

## Decomposition

- [ ] Read `last-scan.json` at launch, when the popover opens (if the file changed) and after every scan; show its age and how long the scan took
- [ ] Summary: the selected total, the found total, a bar by class, class rows with switches for `safe` and `rebuildable` (in memory; they seed the review request), `review` and `user-data` as information
- [ ] Types or the minimum size changed in `settings.json` after the report: the same `ignored` rule is applied to the loaded report at once (ADR §6 item 7)
- [ ] Biggest finds: the top four groups by default-selected size (the `group` labels from `rules --json`); each hands its group to the review seam
- [ ] Buttons: Review… and Clean selected… hand a review request to the review seam; until "Review and clean window" builds the window, the app's router opens nothing and the popover says so
- [ ] Footer: Scan now with the elapsed time in place (through the runner, so a second click joins the running scan); Full Disk Access from the live probe, with Open Privacy Settings when it is missing; the next scheduled scan from a schedule seam, "Scheduled scans off" until "Background scan and notifications in the app" provides the due function; while paused, "Scans paused" with Resume in place of Scan now
- [ ] Moved to "Background scan and notifications in the app", shown through this popover's notice slot: the flag a `--scheduled` instance leaves when notification permission is not yet determined, and the offer to replace the 0.1.0 LaunchAgent
- [ ] States: no report yet (first scan; when the probe says Full Disk Access is missing, it leads with Open Privacy Settings and says that a scan without it asks for each protected folder), scanning, nothing worth cleaning (under the threshold; "Nothing found to clean" when nothing can be freed, with no Biggest finds), report unreadable; notices for a settings error, a busy lock, a failed scan and a missing CLI
- [ ] Status item icon, as artboard 3 with the owner's changes of 2026-10-19 (after the first build):
  - idle: the glyph as the menu bar draws it;
  - busy: an arc turns while any CLI run is in flight (a scan here, a clean and its rescan once "Review and clean window" feeds the same input);
  - something to free: no mark on the icon. The size shows next to the icon when "Show the size you can free next to the icon" is on, only while over the notification threshold. The size is the reclaimable amount of the default selection, the number a notification names;
  - needs attention: a small warning pictogram in the mark's corner. It shows when Full Disk Access is missing, the CLI is missing, a scan failed, or the last report or `settings.json` cannot be read. Each cause lasts as long as its source: the live probe; the CLI, until a relaunch; a failed scan's notice, until it is dismissed or the next scan starts; the report and the settings, until they read. A busy lock (exit 75: a clean started in Terminal) shows its notice and no warning, because it is no failure. With several causes the icon is the same; the VoiceOver text names the first in this order;
  - paused: the whole glyph grey, as Google Drive greys its icon; a scan that was running when Pause came keeps its arc, grey;
  - the attention badge after a notification, if "Show a badge on the menu bar icon" is on, until the Review window opens.

  Precedence: the warning over the badge. A paused icon keeps its mark.
- [ ] Pause and Resume in the right-click menu (owner, 2026-10-19):
  - kept in `UserDefaults` as `ScansPaused`, so a relaunch and a `--scheduled` instance see it;
  - the menu item reads "Pause scans" or "Resume scans";
  - while paused, every Scan now is off and says why: the popover's footer shows "Scans paused" with Resume, the right-click item reads "Scan now (paused)" and is off, and the Review window's Scan again gives way to "Scans paused"; a scheduled launch scans nothing;
  - a scan already running finishes;
  - the rescan after a clean runs while paused: it is part of the clean (`PopoverModel.scanAfterClean()`);
  - Review and its clean stay available: they are the user's own act, behind a confirmation;
  - the 0.1.0 LaunchAgent runs on its own and is not paused (the offer to replace it is in "Background scan and notifications in the app")
- [ ] The two icon preferences in `UserDefaults`: `ShowSizeNextToIcon` off by default (owner, 2026-10-19: next to the icon the reclaimable amount read as a second total beside "Selected to clean"), `ShowMenuBarBadge` on; the Settings story adds their switches
- [ ] Under "Selected to clean", "Frees about X" when the selection's reclaimable amount rounds to another size (findings that share blocks with copies elsewhere), so a notification's amount matches something in the popover

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

No spec covers this story; the design is the ADR "Menu bar app: Swift shell over the ddc CLI" (§1, §2, §5, §6 items 6–7, §7) and artboards 1 and 3 of the design canvas. The shape is the skeleton's: every piece of logic a criterion tests lives in the local package `app/Packages/DDCKit` and runs under `swift test` with no app host; the app target `app/DeepDriveCleaner/` stays a thin AppKit/SwiftUI shell.

**DDCKit** (`app/Packages/DDCKit/Sources/DDCKit/`), in this order:

1. `ScanReport.swift`: add `home`, `scan_roots`, `volumes`, `Totals.ignored_count` / `ignored_bytes`, `Finding.ignored`, `not_selected` (tagged reason), `private_bytes` and `project`, and a computed `reclaimableBytes` (`private_bytes ?? bytes`, as `crates/ddc-core/src/model.rs`). A hand-written `init(from:)` decodes the optional fields with `decodeIfPresent`; `read(_:)` keeps checking `schema_version` first. The skeleton's `Fixtures/scan-report.json`, which has none of the new fields, stays as the 0.1.0-shaped case.
2. `RuleList.swift` (`rules --json`, version first), `SharedSettings.swift` (reads `settings.json`; a missing file gives the defaults; version first; `.malformed` and `.otherSchema` errors; the app never writes it here) and `DataDirectory.swift` (`DDC_DATA_DIR` when set and not empty, else `~/Library/Application Support/DeepDriveCleaner`, as `Env::from_process`; an injectable root).
3. `ReportSummary.swift`, pure: a finding is ignored when its rule is not `ddc.tombstone` and its rule is in `disabled_rules` or its `bytes < min_size` (an exact mirror of `report/mod.rs`, no `switchable` filter); selected when it has no `not_selected` and is not ignored. When `settings.json` cannot be read, the report's own `ignored` and `selected` stand. When the report's own `full_disk_access` is false (a 0.1.0 agent's scan, say), the summary carries "This scan ran without Full Disk Access; part of ~/Library was not read", whatever the probe says now; a report written by another `ddc_version` than the embedded CLI's says so in the subtitle, and a cancelled one says it is incomplete. Class rows in the order safe, rebuildable, review, user-data; the found total is the sum of the class bytes; "Selected to clean" is the selected bytes of the classes switched on; the reclaimable amount is the default selection's `reclaimableBytes` (the CLI's `selected_reclaimable_bytes`), and `reclaimable > 0 && reclaimable >= threshold` decides "over the threshold", as the notify decision does. The top four groups rank by default-selected bytes, whatever the class switches say, leave out groups with nothing selected, break ties by the order of `groups`, and put a rule missing from the list under "Other"; the detail line is "N idle projects of M" when the group's findings carry a `project` (M the group's findings that are not ignored, N the selected ones), else "N items". Class rows: Safe "Caches that refill themselves"; Rebuildable "N idle projects" with "selected of total"; Review "VMs, models, projects without a lock file"; User data "Downloads, media, backups · report only".
4. `Formatting.swift`: sizes exactly as `size.rs::human` (binary units, one decimal, GiB); counts as on the canvas ("5.5 M files"), not as `size.rs::count`; the report's age (refreshed every minute while the app runs) and duration; volume names as `notify.rs::volume_name` (`/` and `/System/Volumes/Data` are the startup disk, `/Volumes/X` is X); the scan scope ("in your home folder" when the only root is home, "in X" for one other root, "in N folders" otherwise); and the next-scan text.
5. `FullDiskAccessProbe.swift` (`open(O_RDONLY)` on a path it is given; the real TCC.db path is built only in the app target), `AppPreferences.swift` (the two icon preferences, the attention badge, `AttentionBadgePending`, and `ScansPaused`, in an injected `UserDefaults`) and `IconState.swift` (pure, from a `busy` input rather than "scanning": inner arc or dot; a mark, the warning (its `Attention` causes) over the badge, and no mark for the threshold (owner, 2026-10-19); `dimmed` while paused; the size label while over the threshold and the preference is on; an accessibility description).
6. `CLIRunner.swift` gains a `CLIRunning` protocol so the model can take a fake; `CLIFailure.swift` turns an outcome and the stderr tail into one sentence (the last `ddc:` line for exit 1, busy for 75, an app bug for 2); `ReviewRouting.swift` holds `ReviewRequest { group, classes, confirm }` and the router protocol, whose `open` returns whether a Review window really opened; `PopoverModel.swift` is a `@MainActor @Observable` class over all of it: `start()` (probe, settings, report, then `rules --json` once before any scan), `popoverWillOpen()` (probe again; reload what changed), `scanNow()` (one scan in flight, its elapsed time counted from the first request; the app's own run is decoded from its stdout, and `last-scan.json` stays the source at launch and when the file changes, which a one-minute check of its modification date notices), and `review` / `cleanSelected` / "See what was found" for the seam. The scanning state is `ScanProgress { started, files: UInt64? }`: the popover writes "N files read · T s" when `files` is present and "T s" until "CLI: progress events" feeds it. `rules --json` runs once per process in normal mode (a `--scheduled` instance that is never promoted does not need it). Reports are decoded off the main actor.

**App target** (`app/DeepDriveCleaner/`): `PopoverView.swift` renders artboard 1 from the model, with semantic colours for dark mode and the class colours of the canvas; the gear opens Settings. `StatusItemController.swift` follows `model.iconState` (variable length with the size as a monospaced-digit title; the arc animates only while a scan runs); `StatusIcon.swift` draws an icon state from `DriveGlyph`, whose default call keeps drawing exactly today's glyph because `scripts/icons/main.swift` builds the AppIcon from it. The icons with no mark stay template images (so the arc is not the canvas's accent colour), drawn at 40 % while paused; the icons with a mark are drawing-handler images that resolve `NSColor.labelColor` (`tertiaryLabelColor` while paused) at draw time, so the glyph follows the menu bar's appearance, with the badge in the accent colour and the warning an orange triangle with a white "!" (owner, 2026-10-19). Scan now keeps the popover open, so the scanning state shows in place; the class switches and the find rows carry accessibility labels. `AppDelegate.swift` creates the model in every launch mode and routes Scan now (popover, right-click menu, the scheduled path) through `model.scanNow()`, and the rescan after a clean through `model.scanAfterClean()`; the router is a stub that logs and returns false until "Review and clean window"; the schedule seam returns nil until "Background scan and notifications in the app"; `DebugMenu.swift` uses the DDCKit probe. Open Privacy Settings tries `x-apple.systempreferences:com.apple.preference.security?Privacy_AllFiles`, then the `com.apple.settings.PrivacySecurity.extension` form, then the Privacy & Security pane; the manual check decides which one Tahoe honours.

**Decided at this gate** (open on the canvas): sizes are GiB everywhere, next to the icon too, as in the CLI and its notifications (the canvas wrote "13.5 GB"); the badge is an accent dot in the mark's corner, on by default (the "!" mark gave way to the warning pictogram on 2026-10-19); finds rank by selected size and their detail lines give counts, because `rules --json` carries no display names; the class switches live in memory and reset at launch; the size next to the icon and the "could be freed" sentence are the reclaimable amount of the default selection, the number the notification names, while "Selected to clean" follows the switches and may differ.

**Checks without dialogs:** `make app-test` and an unsigned `xcodebuild … CODE_SIGNING_ALLOWED=NO`; the owner runs `make app` and opens the app from Finder for the manual check.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Unit test: from a recorded report, with the scan-time `settings.json` and the `rules --json` of the same CLI stored next to it, the view model computes the selected total, the class totals and the top four groups; with the scan-time settings it reproduces the report's own totals; `ignored` findings are left out, and so are findings of a rule switched off, or under a `min_size` raised, after the report was written; a finding whose `bytes` equals `min_size` is kept and one a byte under is left out; a `ddc.tombstone` is never left out; a report without the optional fields decodes, and one with no findings shows "Nothing found to clean" with no Biggest finds
- [ ] Unit test: pressing Scan now twice while a scan runs asks the runner once, the footer shows the running scan with its elapsed time, and after it ends the popover shows the scan's report
- [ ] Unit test: the attention badge is set by the notification-posted event, survives a relaunch, and is cleared only when the review seam reports that the Review window opened; with the badge preference off it is never drawn
- [ ] Unit test: the icon state function covers:
  - idle and scanning;
  - something to free with and without the size, under, at and over the threshold, with no mark in any of them;
  - needs attention for each of its causes;
  - paused (grey), alone, with a mark, and while busy;
  - the badge on and off, the warning over the badge.

  The size label is the reclaimable amount, unchanged by the class switches.
- [ ] Unit test: Pause turns Scan now off with no runner call, survives a relaunch (a new model on the same defaults), and makes a scheduled launch scan nothing; the rescan after a clean still runs; Resume turns Scan now back on
- [ ] Unit test: each warning cause follows its source: a failed scan shows the warning until its notice is dismissed or the next scan starts; a busy lock shows its notice and no warning
- [ ] Unit test: the footer's and the icon's Full Disk Access state follow the probe, not the report: a report with `full_disk_access: true` and a probe that fails shows Full Disk Access as missing; the probe, given a path, reports granted for a readable file and missing for a missing or unreadable one; a report with `full_disk_access: false` and a probe that succeeds shows the footer as granted and the summary's without-access note
- [ ] Unit test: the footer's next scan is the schedule seam's next slot, written as today, tomorrow or a weekday at the hour, and "Scheduled scans off" when there is none
- [ ] Unit test: Review…, a biggest find and Clean selected… hand the review seam the group and the classes switched on, and Clean selected… asks for the confirmation sheet; structural test: no app or DDCKit source except `CLIRunner.swift` names the clean run kind (the token `.clean` as a whole word; "Review and clean window" later narrows it to exactly one caller)
- [ ] Unit test: a scan that exits 1 shows the CLI's last `ddc:` line (a malformed `settings.json`, settings roots that do not exist) and keeps the last report on screen; exit 75 shows that a clean holds the lock; a report of another `schema_version` shows that it could not be read, with Scan now; with no embedded CLI (`HelperLocator` throws) the popover shows the notice, Scan now is disabled, and the last report still shows
- [ ] Manual check (`make app`, the app opened from Finder; before it, Debug → Unregister Scheduler Agent, and Full Disk Access granted to `build/Debug/DeepDriveCleaner.app`, or expect one folder dialog per protected area on the first scan): with no report the popover offers the first scan; with Full Disk Access missing the footer says so and opens the Privacy pane; the icon reads in light and dark menu bars in every state (idle, busy, warning, paused, paused while busy, badge), and the popover reads in dark mode; Pause from the right-click menu greys the icon, turns Scan now off there and in the footer, which shows "Scans paused" with Resume, and Resume restores both

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- The popover never cleans by itself: Clean selected… only opens the confirmation sheet (ADR "Menu bar app: Swift shell over the ddc CLI", §7).
- Totals come from the saved report; the popover does not rescan to open.
- From "CLI: shared settings, types and the notify decision":
  - `ignored` (absent when false), `volumes` (absent when empty) and `Totals.ignored_count` / `ignored_bytes` (absent when 0) are optional in the scan report, so `ScanReport.swift` decodes them with `decodeIfPresent`; a strict decode fails on the first report without them.
  - The home folder's volume reports its mount point as `/System/Volumes/Data`; the popover names it as the startup disk.
  - A malformed `settings.json`, or `roots` naming only folders that do not exist (an unplugged disk), makes every scan exit 1 naming the file, with nothing on stdout. The popover shows that error and never falls back to scanning home.
- The popover never runs `clean`; the clean run kind has one caller, owned by "Review and clean window", and the structural test here keeps everything else away from it.
- The live probe, not the report's `full_disk_access`: the report may come from Terminal's grant or the 0.1.0 agent. Tests probe only temporary paths; nothing in the tests touches TCC.db.
- Mirror the core exactly and pin it with drift tests that read the Rust sources through `#filePath`, as the skeleton's `schemaVersionMatchesTheCLI` does: `TOMBSTONE` (`crates/ddc-core/src/rules/mod.rs`), `RULES_SCHEMA_VERSION` (`rules/listing.rs`), `SETTINGS_SCHEMA_VERSION` (`settings.rs`), `DEFAULT_THRESHOLD` (`crates/ddc-cli/src/cmd/background.rs`) and `volume_name` (`crates/ddc-cli/src/notify.rs`). A Swift `ignored` rule that drifts would show totals the CLI does not clean.
- A foreground `scan` has no skip list (`cmd/scan.rs`; only the background scan skips the prompting areas). Without Full Disk Access on the app, the first Scan now asks for Desktop, Documents, Downloads and other apps' data, one dialog per area, in the app's name. The `--scheduled` path keeps the skeleton's scan, skipped while paused, until "Background scan and notifications in the app" adds the due function.
- `rules --json` waits behind a running scan (every run is serialised), so it runs once at launch before any scan.
- A 0.1.0 LaunchAgent, or the scheduler agent registered from the Debug menu, keeps writing `last-scan.json` until "Background scan and notifications in the app" replaces it; the popover reloads when the file changes.
- Scan progress shows the elapsed time only; file counts arrive with "CLI: progress events", whose app side feeds `ScanProgress.files` (a criterion of that story).

## Dependencies

- [[epics/DDC-002/stories/story-app-skeleton-and-build-pipeline|App skeleton and build pipeline]]
- [[epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision|CLI: shared settings, types and the notify decision]]

## Attachments

- Design canvas: https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3

---

*Last updated: 2026-10-19*

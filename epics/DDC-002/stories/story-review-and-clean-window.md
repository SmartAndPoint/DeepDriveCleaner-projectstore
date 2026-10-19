---
type: story
id: "story-review-and-clean-window"
epic: "DDC-002"
title: "Review and clean window"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-19
external_refs: {design_canvas: "https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3"}
tags: [app, ui, cleanup]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Review and clean window

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

This story is artboards 4 to 6 of the design canvas. The user browses what was found, by group or by item, and selects. A sheet lists every item to confirm, with live warnings. The user then watches progress and sees the result, with what was skipped grouped by cause. It is the only way the app cleans (ADR "Menu bar app: Swift shell over the ddc CLI", §7).

## Decomposition

Rough until the plan gate.

- [ ] Window: a sidebar of classes and groups, and a table of items with:
  - the path under `~` (or the volume), the size and the frees estimate;
  - the project's idle days and whether it has a lock file;
  - notes (`no-lockfile`, `app-owned`, `kept-by-marker`, `excluded`) and the `not_selected` reason
- [ ] Groups / items toggle; search; sort by size, path and idle days
- [ ] Selection: defaults from the report; `review` items start unselected; `user-data` and `ignored` items cannot be selected
- [ ] Reveal in Finder; "Protect this folder" writes `.ddc-keep` after a confirmation
- [ ] Confirmation sheet:
  - counts per class;
  - every item with its action (delete, Trash, the tool's command);
  - for each `review` item, Delete permanently, defaulting to the app preference "Delete review items instead of moving them to the Trash" (off: the Trash);
  - warnings for the Trash and for local snapshots;
  - Cancel and Clean
- [ ] Live warnings: the sheet opens at once and runs `clean --items --dry-run --json` over the listed items (no rescan); the predicted skips stream in per item ("Spotify is running · will be skipped"); Check again reruns it
- [ ] Clean: the sheet's Clean action, and nothing else in the app, runs `clean --items --json --progress json` with exactly the confirmed items (each with its class, action and `permanent`); it shows the preparing phase and item progress; Cancel stops after the current item While the clean and its rescan run, the sheet sets the `busy` input of the icon state of "Menu bar popover", so the arc turns as on artboard 3.
- [ ] Result: deleted, moved to the Trash, tools run, skipped and failed; free space before and after for each volume; skipped items grouped by cause (count and size per cause, the items inside), in plain language; Open journal

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Unit test: the `--items` payload is exactly the confirmed list, each item with its class, action and `permanent`; no item the user did not see in the sheet is sent
- [ ] Unit test: a `review` item's `permanent` starts from the app preference and changing one item changes only that item
- [ ] Structural test: the runner's clean operation has exactly one caller, the sheet's Clean action; notifications, the scheduler and the popover reach only the window or the sheet
- [ ] Unit test: `user-data` and `ignored` findings cannot be selected, and `review` findings start unselected
- [ ] Unit test: the popover's review request ("Menu bar popover": a group, the classes switched on, and whether Clean selected… asked for the sheet) seeds the window's group filter and selection, opens the confirmation sheet when asked, and the router reports the window opened so the attention badge clears
- [ ] Test with a fake CLI (a script that replays recorded dry-run, progress and result JSON): the sheet shows the predicted skips, then the window shows the progress, then the result with skips grouped by cause; every skip reason the executor has maps to text
- [ ] Test with a fake CLI that writes its dry-run lines with a delay: the sheet is shown before the first line arrives, and each warning appears on its item as its line arrives
- [ ] Manual check on the reference machine: cleaning a few selected `safe` items from the window frees space, the before and after values are shown, and the journal lists exactly those items
- [ ] Manual check: Cancel during a clean stops after the current item, and the result lists what was done

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- From "CLI: clean an explicit item list":
  - `clean --items --json` exits 0 whenever it produced a report, failed items included. After a cancel it ends by the signal (`.cancelled`) with the report on stdout. The app decodes stdout in both cases.
  - The CLI installs its handler only after the rescan is saved. A cancel before that ends it by the signal with empty stdout. A cancel any later gives a report, possibly with every item `not-run`. So on `.cancelled` the app decodes stdout when it is not empty, and otherwise shows "cancelled, nothing was cleaned".
  - The CLI saves its rescan before executing, so the popover rescans after every clean run, cancelled ones included.
  - What was left undone is read from the report, never from how the process ended: `cancelled`, the `not-run` items, and an interrupted tool, which shows as a `native` outcome whose command has `success: false`. A signal that arrives after the last item started ends the CLI by the signal with a complete report and `cancelled: false`.
  - The dry run streams one JSON line per item on stdout. `CLIRunner` (`app/Packages/DDCKit/Sources/DDCKit/CLIRunner.swift`) returns stdout only at EOF and streams only stderr lines, so showing each warning as its line arrives needs a stdout line callback.
  - The dry run takes the shared lock and exits 75 (`.busy`) while a terminal `ddc clean` holds it, so the sheet needs a busy state. Exit 2 (`.refused`) on a list the app built is an app bug.
- From "CLI: shared settings, types and the notify decision": the app passes no positional roots and no `--exclude` to `scan`, `clean --items` or its dry run; all three resolve them from `settings.json`. A settings change between review and clean can turn items into `changed-since-scan` or `kept`; the result screen shows them. `--items` with an `--exclude` that drops a protected folder exits 2. `ignored` (absent when false), `volumes` (absent when empty) and `Totals.ignored_count` / `ignored_bytes` (absent when 0) are optional in the scan report, so `ScanReport.swift` decodes them with `decodeIfPresent`; a strict decode fails on the first report without them.
- Skip reasons come from the executor (`SkipReason`), so their wording lives in one table in the app with a test that covers every variant.
- The rescan before cleaning (about 40 s on the reference machine) shows as the first phase of the progress.
- A dry run predicts; the clean re-checks everything again, so a prediction never replaces a check.

## Dependencies

- [[epics/DDC-002/stories/story-app-skeleton-and-build-pipeline|App skeleton and build pipeline]]
- [[epics/DDC-002/stories/story-cli-clean-an-explicit-item-list|CLI: clean an explicit item list]]
- [[epics/DDC-002/stories/story-cli-progress-events|CLI: progress events]]
- [[epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision|CLI: shared settings, types and the notify decision]]

## Attachments

- Design canvas: https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3

---

*Last updated: 2026-10-19*

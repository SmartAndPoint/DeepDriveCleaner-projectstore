---
type: story
id: "story-menu-bar-popover"
epic: "DDC-002"
title: "Menu bar popover"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-19
external_refs: {design_canvas: "https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3"}
tags: [app, ui]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Menu bar popover

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
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

Rough until the plan gate.

- [ ] Read `last-scan.json` at launch and after every scan; show its age
- [ ] Summary: the selected total (without `ignored` findings), the found total, a bar by class, class rows with switches for `safe` and `rebuildable` (the popover's choice seeds the Review window), `review` and `user-data` as information
- [ ] Types or the minimum size changed in Settings after the report: the same `ignored` rule is applied to the loaded report at once (ADR §6 item 7)
- [ ] Biggest finds: the top four groups by size (the `group` labels from `rules --json`); each opens the Review window filtered to it
- [ ] Buttons: Review…, and Clean selected…, which opens the confirmation sheet of "Review and clean window" with the popover's selection
- [ ] Footer: Scan now with progress in place (through the runner, so a second click joins the running scan); Full Disk Access from the live probe (whether the app can open TCC.db), not the report's `full_disk_access`; the next scheduled scan from the scheduler's due function
- [ ] The flag a `--scheduled` instance leaves when notification permission is not yet determined: the next popover open shows it, with a button that asks for the permission (story "Background scan and notifications in the app")
- [ ] States: no report yet (first scan), scanning, nothing worth cleaning (under the threshold), report unreadable
- [ ] Status item icon, as artboard 3:
  - idle;
  - scanning (an arc turns while a scan or clean runs);
  - something to free: a dot, and the size next to the icon when Settings → "Show the size you can free next to the icon" is on, only while over the notification threshold;
  - needs Full Disk Access (!), from the live probe;
  - the attention badge after a notification, if "Show a badge on the menu bar icon" is on, until the Review window opens
- [ ] On first launch with the 0.1.0 LaunchAgent present, the offer to replace it (its flow is in "Background scan and notifications in the app")

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Unit test: from a recorded report, the view model computes the selected total, the class totals and the top four groups; `ignored` findings are left out, and so are findings of a rule switched off after the report was written
- [ ] Unit test: pressing Scan now twice while a scan runs asks the runner once, and the button shows the running scan's progress
- [ ] Unit test: the attention badge is set by a posted notification and cleared when the Review window opens
- [ ] Unit test: the icon state function covers idle, scanning, something to free (with and without the size, under and over the threshold), needs Full Disk Access, and the badge on and off
- [ ] Unit test: the footer's and the icon's Full Disk Access state follow the probe, not the report: a report with `full_disk_access: true` and a probe that fails shows Full Disk Access as missing
- [ ] Unit test: "Next scan" equals the scheduler's due function for a daily and a weekly schedule, before and after the slot, and after a missed slot; a Scan now does not move it
- [ ] Manual check: with no report, the popover offers the first scan; with Full Disk Access missing, the footer says so and opens the Privacy pane

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

## Dependencies

- [[epics/DDC-002/stories/story-app-skeleton-and-build-pipeline|App skeleton and build pipeline]]
- [[epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision|CLI: shared settings, types and the notify decision]]

## Attachments

- Design canvas: https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3

---

*Last updated: 2026-10-19*

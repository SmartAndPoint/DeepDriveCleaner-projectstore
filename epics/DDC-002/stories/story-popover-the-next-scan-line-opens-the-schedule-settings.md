---
type: story
id: "story-popover-the-next-scan-line-opens-the-schedule-settings"
epic: "DDC-002"
title: "Popover: the Next scan line opens the Schedule settings"
status: planned
priority: p3
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [app, popover, settings, schedule]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Popover: the Next scan line opens the Schedule settings

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p3 |
| **Assignee** | Evgenii Konev |

---

## Description

Requested by the owner on 2026-10-28. The popover's footer ends with the next scheduled scan, for example "Next scan tomorrow, 11:00" (`PopoverView.footer`, text from `PopoverModel.nextScanText`). Today it is plain text. A person who reads it and wants another time has to find Settings → Schedule by hand.

The line becomes a link-style button. A click closes the popover and opens the Settings window on the Schedule pane, through `SettingsWindowController.show(pane: .schedule)`, which already exists. The text stays as it is, in the secondary colour set on the button itself (not inherited from the footer), with the pointing-hand cursor on hover. The tooltip and the accessibility hint follow the destination: "Change the schedule in Settings" / "Opens Settings, Schedule", or "Replace the old background scan in Settings" / "Opens Settings, Permissions".

What the line says, and where a click goes (`Wording.nextScan`):
- "Next scan today, 13:00", "Next scan tomorrow, 11:00", "Next scan Monday, 10:00", "Next scan within the hour", "Scheduled scans off": Settings → Schedule.
- "Replace the old background scan first", while the 0.1.0 agent runs: Settings → Permissions, where its "Replace It" button lives.
- While scans are paused the footer's left side shows "Scans paused" with Resume, and the line is not shown. That stays.

It opens a window and changes nothing. No scan starts and no setting changes from the click.

## Decomposition

- [ ] DDCKit: `enum SettingsDestination { case schedule, permissions }` beside `NextScan` in `ScanDue.swift`, and `NextScan.settingsDestination`: `.at` and `.off` → `.schedule`, `.replaceOldScanFirst` → `.permissions`. The tooltip and hint strings live with it, so they cannot drift from the route.
- [ ] `PopoverView` gets `openNextScan: (SettingsDestination) -> Void` beside `openSettings`. The footer's Next scan text becomes a `Button` with the link style, the secondary colour set explicitly, the tooltip and the hint; its action calls `openNextScan(model.nextScanValue.settingsDestination)` and nothing else.
- [ ] `StatusItemController` wraps it as it wraps `openSettings`: `popover.performClose(nil)`, then the handler. `AppDelegate` maps the destination to `SettingsWindowController.Pane` and calls `settings?.show(pane:)`. The menu's "Settings…" item keeps the pane-less `openSettings`.

## Implementation Plan

## Acceptance Criteria

- [ ] Test (DDCKit, in `PopoverModelTests`): `settingsDestination` is `.schedule` for `.at(date)` (a future time and one already due) and for `.off`, and `.permissions` for `.replaceOldScanFirst`; the tooltip and hint name the same pane.
- [ ] Contract test (source check, in `ContractTests`' style): `PopoverView.swift` has one button whose label is `model.nextScanText`, and its action calls only `openNextScan`.
- [ ] While scans are paused, the line is not shown, as today.
- [ ] Manual (owner): clicking "Next scan …" closes the popover and opens Settings → Schedule, both when Settings was closed and when it was open on another pane.

## Final Summary

## Dependencies

- "Menu bar popover" (DDC-002): the footer
- "Settings: folders, types, notifications" (DDC-002): the Schedule pane and `show(pane:)`

## Attachments

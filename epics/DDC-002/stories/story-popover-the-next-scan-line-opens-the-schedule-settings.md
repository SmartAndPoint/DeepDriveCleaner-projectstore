---
type: story
id: "story-popover-the-next-scan-line-opens-the-schedule-settings"
epic: "DDC-002"
title: "Popover: the Next scan line opens the Schedule settings"
status: done
priority: p3
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-30
external_refs: {}
tags: [app, popover, settings, schedule]
code_refs: ["app/DeepDriveCleaner/AppDelegate.swift", "app/DeepDriveCleaner/PopoverView.swift", "app/DeepDriveCleaner/SettingsWindowController.swift", "app/DeepDriveCleaner/StatusItemController.swift", "app/Packages/DDCKit/Sources/DDCKit/ScanDue.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ContractTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/PopoverModelTests.swift"]
specs: []
started_at: "2026-10-29T09:31:57.831Z"
closed_at: "2026-10-30T06:45:42.341Z"
plan_updated_at: "2026-10-29T09:31:57.831Z"
---

# Popover: the Next scan line opens the Schedule settings

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
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

- [x] DDCKit: `enum SettingsDestination { case schedule, permissions }` beside `NextScan` in `ScanDue.swift`, and `NextScan.settingsDestination`: `.at` and `.off` → `.schedule`, `.replaceOldScanFirst` → `.permissions`. The tooltip and hint strings live with it, so they cannot drift from the route.
- [x] `PopoverView` gets `openNextScan: (SettingsDestination) -> Void` beside `openSettings`. The footer's Next scan text becomes a `Button` with the link style, the secondary colour set explicitly, the tooltip and the hint; its action calls `openNextScan(model.nextScanValue.settingsDestination)` and nothing else.
- [x] `StatusItemController` wraps it as it wraps `openSettings`: `popover.performClose(nil)`, then the handler. `AppDelegate` maps the destination to `SettingsWindowController.Pane` and calls `settings?.show(pane:)`. The menu's "Settings…" item keeps the pane-less `openSettings`.

## Implementation Plan

From the planner's pass of 2026-10-29.

1. `ScanDue.swift`: `SettingsDestination` with `help` and `hint`, and `NextScan.settingsDestination` by an exhaustive switch. Test in `PopoverModelTests`.
2. `PopoverView.swift`: `openNextScan`, and the footer's `Button(model.nextScanText)` inside `if !model.paused`, `.buttonStyle(.link)` then `.foregroundStyle(.secondary)`, `.pointerStyle(.link)`, `.help`, `.accessibilityHint`.
3. `StatusItemController.swift`: the wrapper, `performClose(nil)` then the handler; the menu's "Settings…" keeps `openSettings`.
4. `SettingsWindowController.swift`: `Pane.init(_ destination:)`. `AppDelegate.swift`: `settings?.show(pane: .init(destination))`.
5. `ContractTests`: the one button, its one call, its modifiers, the paused guard, the wrapper and the pane-less menu item.
6. Gate, unsigned build, `projectstore:reviewer`.

## Acceptance Criteria

- [x] Test (DDCKit, in `PopoverModelTests`): `settingsDestination` is `.schedule` for `.at(date)` (a future time and one already due) and for `.off`, and `.permissions` for `.replaceOldScanFirst`; the tooltip and hint name the same pane. — evidence: `PopoverModelTests.nextScanDestination`
- [x] Contract test (source check, in `ContractTests`' style): `PopoverView.swift` has one button whose label is `model.nextScanText`, and its action calls only `openNextScan`. — evidence: `ContractTests.nextScanLink`
- [x] Contract test: while scans are paused, the line is not shown, as today (the button sits inside `if !model.paused`). — evidence: `ContractTests.nextScanLink` (anchored to `if !model.paused {`)
- [x] Contract test: `StatusItemController` closes the popover before it calls the handler, the menu's "Settings…" stays pane-less, and `AppDelegate` opens the pane with `show(pane:)`. — evidence: `ContractTests.nextScanLink`; the app compiles unsigned (`xcodebuild … CODE_SIGNING_ALLOWED=NO build`)
- Manual checks (1) moved to [Owner checks: manual acceptance across stories](../../../ops/owner-checks-manual-acceptance-across-stories.md) on 2026-10-30, by the owner's decision.

## Final Summary

**What changed** (code commit a6adf01, released in 0.6.0 with 8af8cf1):
- The popover footer's "Next scan …" line is a link. It closes the popover and opens Settings on the pane `NextScan.settingsDestination` names: Schedule, or Permissions while the old background scan must be replaced first.
- The menu's "Settings…" stays pane-less. The line is still hidden while scans are paused.

**Tests run:** `PopoverModelTests.nextScanDestination`, `ContractTests.nextScanLink`; an unsigned app build (`CODE_SIGNING_ALLOWED=NO`).

**Remaining risk:** one manual check moved to the owner list: the click in the real popover, with Settings closed and with Settings open on another pane.

## Dependencies

- "Menu bar popover" (DDC-002): the footer
- "Settings: folders, types, notifications" (DDC-002): the Schedule pane and `show(pane:)`

## Attachments

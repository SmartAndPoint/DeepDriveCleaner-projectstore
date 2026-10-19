---
type: story
id: "story-settings-folders-types-notifications"
epic: "DDC-002"
title: "Settings: folders, types, notifications"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-19
external_refs: {design_canvas: "https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3"}
tags: [app, ui, settings]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Settings: folders, types, notifications

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

This story is artboard 7 of the design canvas: the General, Folders, Types, Notifications and Permissions tabs.

What the CLI also reads is stored in `settings.json` (story "CLI: shared settings, types and the notify decision"), so a scan from the terminal and a scan from the app use the same folders and types. Preferences only the app uses stay in its `UserDefaults`.

## Decomposition

Rough until the plan gate.

- [ ] General:
  - Open at login (state from `SMAppService.status`);
  - Scan: daily, weekly or off, at a whole hour ("Daily at 10:00"), the hour the scheduler's hourly tick can match; off unregisters the scheduler agent;
  - "Show the size you can free next to the icon";
  - "Delete review items instead of moving them to the Trash", off by default
- [ ] Folders: "Scan these folders" (default: the home folder; external volumes such as `/Volumes/Work` allowed), added and removed with a folder picker; "Never touch these folders" (excludes) and how `.ddc-keep` does the same; a root inside another root is refused
- [ ] Types: one switch per rule group from `rules --json`, with its class, written as `disabled_rules`; "Projects count as idle after N days"; "Hide items smaller than N MB" (`min_size`). A change applies to the loaded report at once and to the next scan through `settings.json`
- [ ] Notifications: the threshold, with the repeat rule (a fifth more, or a week) explained; "Also when the disk is almost full" (`notify_low_disk`), explained as notifying at every scheduled scan while the disk stays almost full; "Show a badge on the menu bar icon"; "Only notify during working hours" (9:00 to 19:00, weekdays)
- [ ] Permissions: Full Disk Access state from the live probe, as the popover (not the report's `full_disk_access`), with Open Privacy Settings; notification permission state; "Old background scan found · Replace it" while the 0.1.0 LaunchAgent exists (the flow is in "Background scan and notifications in the app")
- [ ] `settings.json` is written atomically and used by the next scan

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test: switching a type off writes its rule ids to `disabled_rules`, the loaded report marks those findings ignored at once, and the next scan marks them `ignored`; the same for the minimum size
- [ ] Test with a fake CLI: roots, external-volume roots and excludes edited in Settings reach the next scan through `settings.json`; the app passes no flags for them
- [ ] Test: a folder inside an existing root, or the same folder twice, is refused with a message
- [ ] Unit test: the schedule offers whole hours only, and the next scan it shows equals the scheduler's due function
- [ ] Unit test: the Full Disk Access probe reports granted when a TCC.db path it is given can be opened and missing when it cannot (an injectable path, as `report::full_disk_access` reads `~/Library/Application Support/com.apple.TCC/TCC.db`)
- [ ] Unit test: app preferences (size next to the icon, badge, working hours, the default for `review` items) persist in `UserDefaults` and never appear in `settings.json`

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- The groups come from the CLI (`rules --json`), so a new rule appears in the right group without an app release.
- `disabled_rules` and `min_size` hide findings from selection and totals; they do not stop the scanner from walking those folders.
- The canvas's help under Open at login says it keeps "the daily scan running". With the scheduler agent (ADR §5) scans run without the login item; the help text says the icon only.

## Dependencies

- [[epics/DDC-002/stories/story-app-skeleton-and-build-pipeline|App skeleton and build pipeline]]
- [[epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision|CLI: shared settings, types and the notify decision]]
- [[epics/DDC-002/stories/story-background-scan-and-notifications-in-the-app|Background scan and notifications in the app]]

## Attachments

- Design canvas: https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3

---

*Last updated: 2026-10-19*

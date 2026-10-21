---
type: story
id: "story-settings-folders-types-notifications"
epic: "DDC-002"
title: "Settings: folders, types, notifications"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-21
external_refs: {design_canvas: "https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV"}
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

### Revision after the owner's review (2026-10-21)

Artboard 10 of the design canvas ("Settings, revised") replaces artboard 7. Its tabs are General, Folders, Schedule, Automation, Types and Permissions. The old tabs map to the new ones:
- General's Scan row moves to Schedule.
- Notifications (threshold, low disk, working hours) moves to Automation, under "Notify me when".
- The badge and the size next to the icon stay in General.

The Decomposition and ACs below still describe artboard 7 until the plan gate re-cuts them.

The window is an `NSTabViewController` with `tabStyle = .toolbar`, and the window title is the pane's name. A SwiftUI `TabView` in an `NSWindow` would draw segments, not toolbar tabs.

The owner decided four things.

**1. Automatic cleaning, strict variant.** "After a scheduled scan" offers three choices:
- Notify me (the default);
- Clean download caches automatically, then notify me;
- Only show it in the menu bar.

The automatic choice has these limits:
- **What it cleans.** Only `safe` groups marked Auto in Types. Auto is on by default only for caches a package manager downloads again: npm, pnpm, yarn, bun, pip, Poetry, Homebrew and CocoaPods.
- **Groups have to split first.** Today `rules --json` puts all of these in one group with Cargo, uv, Go, Gradle (`crates/ddc-core/src/rules/listing.rs`, `GROUPS`), and Types has one switch per group. So `GROUPS` splits that group into "Package downloads" and "Toolchain caches" (Cargo, uv, Go modules and build cache, Gradle), as artboard 10 draws them. Auto is then stored per group in `settings.json`.
- **What stays off by default.** These are `safe` too:
  - Toolchain caches, which an offline build needs;
  - IDE caches, because of re-indexing;
  - Xcode builds and project build caches, which are rebuilt by compute;
  - browser and app caches: they carry `owners`, so a running browser makes them skip as in use most of the day.
- **When it runs.** Only when it frees at least N GiB, counted as private bytes. Anything in use is skipped, as in 0.2.1.

Before any code it needs amendments:
- the safety ADR: §8 ("the background path contains no call to the cleanup executor") and its Alternative A;
- the menu bar ADR: §7 (the sheet's Clean is the only caller of a clean, which `ContractTests.cleanHasOneOwner` enforces) and §5 (a scheduled instance lives about a minute).

It also needs:
- how a promoted scheduled instance shows the running clean with Stop;
- the "cleaned" notification's category and what `notify ack` records;
- the rescan after the clean.

It has its own story: "Automatic cleaning of download caches after a scheduled scan" (DDC-002). This story only draws the Automation tab and stores the choice.

**2. Exclude with two modes.**
- "Count, never clean" is today's `excludes` (`user.kept`, still read and sized).
- "Skip entirely" is new: the folder is not read. It needs:
  - an additive optional `skips` in `settings.json`;
  - a `--skip` flag;
  - the report's skipped reason, so Review says "not scanned";
  - a line in safety ADR §2 item 1.

  The `--exclude` confirmation of menu bar ADR §6 item 6 does not apply to a skip.

  This story owns that CLI change. `ContractTests.settingsKeys` checks the new key against `Settings`.

**3. Protected folders, simplified.** There is no per-folder Allow…; a scheduled scan without Full Disk Access skips every prompting area anyway (`crates/ddc-cli/src/cmd/background.rs`).
- **Without Full Disk Access,** Folders shows one banner with Open Privacy Settings. Below it is a "Not readable" list: the folders the last foreground scan reported in `inaccessible`, each with Open Privacy Settings.
- **With Full Disk Access,** only a confirmation line.
- **iCloud Drive** is never walked, so it is never listed.

**4. Schedule tab.** It holds:
- Scan automatically;
- Daily or Weekly, with the day and a whole hour;
- Pause, the same `ScansPaused` as the popover;
- Next scan, from the due function of "Background scan and notifications in the app": "Today" or "Tomorrow" at the slot, and "Within the hour" after a missed slot or a Resume.

The decomposition below is re-cut at the plan gate.

## Decomposition

Rough until the plan gate.

- [ ] General:
  - Open at login (state from `SMAppService.status`);
  - Scan: daily, weekly or off, at a whole hour ("Daily at 10:00"), the hour the scheduler's hourly tick can match; off unregisters the scheduler agent; while scans are paused from the menu ("Menu bar popover") the row says so, and the schedule is kept for Resume;
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
- [ ] Unit test: the Full Disk Access probe reports granted when a TCC.db path it is given can be opened and missing when it cannot (an injectable path, as `report::full_disk_access` reads `~/Library/Application Support/com.apple.TCC/TCC.db`) — the probe of "Menu bar popover" (`FullDiskAccessProbe`), which Settings reuses rather than adding a second one
- [ ] Unit test: app preferences (size next to the icon, badge, working hours, the default for `review` items) persist in `UserDefaults` and never appear in `settings.json`

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- "Menu bar popover" stores the two icon preferences in `UserDefaults` as `ShowSizeNextToIcon` (off by default) and `ShowMenuBarBadge` (on by default); the General tab's switches write those keys.
- The groups come from the CLI (`rules --json`), so a new rule appears in the right group without an app release.
- `disabled_rules` and `min_size` hide findings from selection and totals; they do not stop the scanner from walking those folders.
- The canvas's help under Open at login says it keeps "the daily scan running". With the scheduler agent (ADR §5) scans run without the login item; the help text says the icon only.

## Dependencies

- [[epics/DDC-002/stories/story-app-skeleton-and-build-pipeline|App skeleton and build pipeline]]
- [[epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision|CLI: shared settings, types and the notify decision]]
- [[epics/DDC-002/stories/story-background-scan-and-notifications-in-the-app|Background scan and notifications in the app]]

## Attachments

- Design canvas: https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV

---

*Last updated: 2026-10-21*

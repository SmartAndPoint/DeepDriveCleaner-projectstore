---
type: epic
id: "DDC-002"
title: "Menu bar app"
status: in-progress
priority: p2
created: 2026-10-18
updated: 2026-10-20
external_refs: {design_canvas: "https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3"}
tags: [app, macos, swift]
code_refs: ["app/", "scripts/build-app.sh", "scripts/icons/", "crates/ddc-cli/", "crates/ddc-core/", "docs/clean.schema.json", "docs/scan.schema.json", "docs/notify.schema.json", "docs/settings.schema.json", "docs/rules.schema.json", "README.md"]
review_status: reviewed
reviewed_at: 2026-10-18
---

# DDC-002: Menu bar app

| Field | Value |
|---|---|
| **Status** | in-progress |
| **Priority** | p2 |
| **Created** | 2026-10-18 |
| **Updated** | 2026-10-18 |

---

## Goal

`DeepDriveCleaner.app` lives in the menu bar.

- **One click** shows what the latest scan found.
- **Review and clean.** The user reviews by group or by item and cleans after a confirmation sheet that lists every item.
- **Settings** choose which folders are scanned and which kinds of objects may be cleaned.
- **Notifications** say when a lot can be freed or the disk is almost full, with the actions Review and Remind me tomorrow.
- **One Full Disk Access grant**, to the app, by bundle id, covers manual and scheduled scans and survives `brew upgrade`.

It ships as the `deepdrivecleaner` cask 0.2.0, for macOS 26, signed by team 8MSDJTBW2D. The one cask also keeps the `ddc` and `deepdrivecleaner` commands on the `PATH`.

## Context

DDC-001 shipped the terminal tool 0.1.0 as a signed, notarized cask. Two things in it point to an app:

- macOS keys the CLI's Full Disk Access by its path, and every cask version has a new path, so the grant is lost on each upgrade.
- Notifications sent through `osascript` have no actions and do not carry the app's name.

The owner asked on 2026-10-18 for a menu bar app in the spirit of GrammarHelper. The how is decided in the ADR "Menu bar app: Swift shell over the ddc CLI":

- a Swift shell (an `NSStatusItem` and an `NSPopover` hosting SwiftUI);
- the CLI bundled inside the app and run as a child process with JSON in and out;
- scheduled scans through a launch agent, registered from the app's bundle, that opens the app;
- notifications posted by the app, with the notify rule kept in the CLI;
- the app's confirmation sheet as the only path to cleaning.

The screens are modelled in the design canvas before any code.

## Stories

| Story | Status | Description |
|-------|--------|-------------|
| [story-app-skeleton-and-build-pipeline](./stories/story-app-skeleton-and-build-pipeline.md) | in-progress | Debug spike tools (Full Disk Access probe, agent registration, test notification); Xcode project for macOS 26, app icon and menu bar glyph, status item and popover shell, one runner that serialises CLI runs, CLI in `Contents/Helpers`, login item, `make app` / `make app-dist` |
| [story-cli-clean-an-explicit-item-list](./stories/story-cli-clean-an-explicit-item-list.md) | done | `clean --items` with class, action and per-item `permanent`; `--json` report; `--dry-run` preflight; SIGTERM for clean with native-child forwarding; exit codes 0 / 1 / 2 / 75 |
| [story-cli-progress-events](./stories/story-cli-progress-events.md) | planned | `--progress json` events for scan and clean; needs the counter of DDC-001 "Progress output while ddc clean runs" |
| [story-cli-shared-settings-types-and-the-notify-decision](./stories/story-cli-shared-settings-types-and-the-notify-decision.md) | done | `settings.json`, `ignored` findings (types, minimum size), `rules --json`, `volumes`, `--notify json`, `notify ack` / `notify snooze`, `schedule install` and the app's scheduler record |
| [story-menu-bar-popover](./stories/story-menu-bar-popover.md) | in-progress | Totals by class, biggest finds, Scan now, Full Disk Access and next-scan status, empty / scanning / nothing-to-clean states, icon states with the size and the badge |
| [story-review-and-clean-window](./stories/story-review-and-clean-window.md) | in-progress | Group and item views, selection, confirmation sheet with dry-run warnings and per-item permanent delete, progress, result with skips grouped by cause |
| [story-review-window-sorted-sidebar-select-all-and-folder-groups](./stories/story-review-window-sorted-sidebar-select-all-and-folder-groups.md) | in-progress | Sidebar tinted by class and ordered by size, Select all, folder subgroups that collapse, no repeated words on rows (design canvas board 9) |
| [story-user-data-pick-your-own-files-for-the-trash](./stories/story-user-data-pick-your-own-files-for-the-trash.md) | planned | `scan` lists entries inside user-data folders; `clean --items` takes a picked entry to the Trash only; User data section and "Your own files" block in the app (safety ADR amendment 2026-10-20) |
| [story-settings-folders-types-notifications](./stories/story-settings-folders-types-notifications.md) | planned | General, Folders (external volumes, never-touch folders), Types (rule groups, idle days, minimum size), Notifications (low disk, badge, working hours), Permissions (Replace it) |
| [story-background-scan-and-notifications-in-the-app](./stories/story-background-scan-and-notifications-in-the-app.md) | planned | Scheduler agent and `--scheduled` mode, notifications with Review / Remind me tomorrow, working hours, offered migration from the 0.1.0 LaunchAgent, Login Items under the app |
| [story-distribution-the-020-cask-with-the-app](./stories/story-distribution-the-020-cask-with-the-app.md) | planned | Cask 0.2.0 (`depends_on macos: :tahoe`, `app` and `binary` from the bundle, `zap launchctl:` for both agents), release tooling for the zipped app, README, upgrade of the reference machine |
| [story-signing-team-parameterise-teamid-then-switch-to-smartandpoint](./stories/story-signing-team-parameterise-teamid-then-switch-to-smartandpoint.md) | planned | Later and independent: one TEAM_ID setting, then the SmartAndPoint organization team; costs every user one Full Disk Access re-grant and renames the developer in Login Items |

## Expected Results

- [ ] `brew upgrade --cask deepdrivecleaner` takes a 0.1.0 user to the app; `ddc` keeps working from the `PATH`
- [ ] One click shows the selected and found totals from the last scan without rescanning
- [ ] Cleaning happens only through the confirmation sheet, and the CLI's executor re-checks every item as in DDC-001
- [ ] One Full Disk Access grant, to the app, covers manual and scheduled scans and survives upgrades
- [ ] Notifications carry Review and Remind me tomorrow and show the app's name and icon
- [ ] Login Items & Extensions lists the scheduled scan under DeepDriveCleaner with the app's icon; the 0.1.0 LaunchAgent is gone after the user accepts the migration
- [ ] The app and the CLI read the same `settings.json`, and switched-off types are never selected

## Dependencies

- [[epics/DDC-001/epic|DDC-001: Disk space cleaner CLI for developers]]: the core, the executor and the release tooling
- [[epics/DDC-001/stories/story-progress-output-while-ddc-clean-runs|Progress output while ddc clean runs]] (raised to p2): the counter "CLI: progress events" emits; a hard dependency of that story
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]

## Decisions

The owner decided the epic's open questions on 2026-10-18; the ADR "Menu bar app: Swift shell over the ddc CLI" records them.

- **Minimum macOS 26**, as GrammarHelper: current OS first. The cask says `depends_on macos: :tahoe` (ADR §3).
- **0.2.0 is signed by the existing team 8MSDJTBW2D**: a proper app first. "Signing team: parameterise TEAM_ID, then switch to SmartAndPoint" is a later, independent story, not a predecessor of the release. Its cost: every user grants Full Disk Access once more, because the designated requirement includes the team; and the developer name in Login Items and the background-activity banner changes to the organization's (ADR §8).
- **The notification's second action is Remind me tomorrow**, as on the canvas. It snoozes for 24 hours; the first scheduled scan after the snooze notifies if still over the threshold, even when the repeat rule would hold it back. The snooze is kept in `notify-state.json`, written only by the CLI at the app's request (`notify snooze`), like the acknowledgement (ADR §5).
- **One cask** for the CLI and the app (ADR §8).
- **The access check runs on a clean machine after the release** (2026-10-18). The app is finished on the reference machine first and dogfooded there, as GrammarHelper is, with Full Disk Access granted to DeepDriveCleaner.app and the app started from Finder, never from Terminal. Then 0.2.0 goes out through the cask, and the owner checks every access question on the work laptop: the grant belongs to the app; the CLI's console use is a capability, not the main path. That check is a criterion of "Distribution: the 0.2.0 cask with the app" and replaces the skeleton's spike on a fresh user account. The risk accepted: if a scheduled scan reports `full_disk_access: false` with the grant on the app, ADR item 5 and Rationale 1 are revised after the scheduling stories are built; the ADR has no fallback for that. Whether `SMAppService` accepts the `open` program, the question the `--tick` fallback answers, is settled earlier, on the reference machine.
- **Later decisions (2026-10-19)**, from "CLI: shared settings, types and the notify decision":
  - an `--exclude` that drops a folder `settings.json` protects is confirmed with y/N naming those folders, and refused where nobody can answer (ADR §6 item 6);
  - low disk fires for a volume only when something selected lies on it, any amount (§6 item 9);
  - low disk is not held back by the repeat rule: every scheduled scan notifies while the disk stays almost full, until it is cleaned or snoozed, and its acknowledgement is the threshold's repeat baseline (§5, §6 item 9).
- **Later decisions (2026-10-19)**, from the owner's first try of the app ("Menu bar popover", "Review and clean window"):
  - the status icon has no threshold dot; a warning pictogram shows what needs the user, and the badge stays (ADR §2);
  - Pause scans and Resume scans in the right-click menu: the icon turns grey, Scan now is off, nothing is due, and the rescan after a clean still runs (ADR §2, §5, §6 item 6);
  - the Review window's Close button returns to the popover.
- **Later decisions (2026-10-20)**, from the owner's try of the Review window:
  - the app icon is option B, "Class ring", from the canvas's app icon board: the drive inside a ring of the class colours (ADR §2; "App skeleton and build pipeline", Implementation Plan, Icons);
  - the Review window gets a sidebar tinted by class and ordered by size, a Select all row, and folder subgroups that collapse ("Review window: sorted sidebar, Select all and folder groups"; design canvas board 9);
  - User data opens in the Review window: the user can pick their own files, and they go to the Trash only ("User data: pick your own files for the Trash"; the safety ADR's amendment of 2026-10-20).

## Open Questions

- [x] Technical, decided at the plan gate of "CLI: clean an explicit item list" (2026-10-18): the full rescan before `--items` stays. Only a whole-tree walk sees the toolchain pins (every pin file and sibling version) and the ancestor claims (a rule that would now claim a directory above an item); a scoped walk would need a second classification path. The per-item checks serve the dry run, which predicts and never replaces the rescan.

## Related

- Design canvas: https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3
- GrammarHelper (the sibling menu bar app): `App/AppDelegate.swift`, `App/SettingsManager.swift`, `Makefile`, `scripts/publish-release.sh`

---

*Last updated: 2026-10-20*

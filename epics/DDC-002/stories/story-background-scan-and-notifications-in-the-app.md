---
type: story
id: "story-background-scan-and-notifications-in-the-app"
epic: "DDC-002"
title: "Background scan and notifications in the app"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-18
external_refs: {design_canvas: "https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3"}
tags: [app, notifications, schedule]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Background scan and notifications in the app

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

Scheduled scans and notifications move from the 0.1.0 LaunchAgent into the app (ADR "Menu bar app: Swift shell over the ddc CLI", §5):
- a launch agent registered from the app's bundle opens the app once an hour, and the app scans when one is due;
- notifications carry the app's name and icon, and the actions Review and Remind me tomorrow;
- the 0.1.0 agent is replaced only when the user agrees.

This story is artboards 2 and 3 of the design canvas, and the migration from 0.1.0.

## Decomposition

Rough until the plan gate.

- [ ] Scheduler agent, self-healing (ADR §5):
  - at every launch with scheduled scans on, call `register()` for `com.smartandpoint.deepdrivecleaner.scheduler` (idempotent), which also recovers an agent approved later or dropped when `brew` replaced the bundle;
  - write the `scheduler` record to `settings.json` whenever the status is `.enabled`; clear it only when scheduled scans are off (and unregister) or registration fails; `.requiresApproval` leaves it as it is;
  - the Settings state follows `SMAppService.status`
- [ ] Due function, shared with the popover and Settings:
  - the schedule (daily or weekly, a whole hour) and the time of the last **scheduled** scan, kept in `UserDefaults`, not `last-scan.json`;
  - a slot missed while the Mac slept is due at the next tick;
  - a Scan now or a terminal `ddc scan` neither moves the slot nor suppresses that day's notification;
  - a pending reminder scan (Remind me tomorrow with a weekly schedule) is due at the first tick after the snooze ends
- [ ] `--scheduled` launch: no status item; scan if due, post a pending notification if one is waiting, quit. A reopen event while running normally: scan if due. A user launch during a scheduled scan promotes the instance (story "App skeleton and build pipeline")
- [ ] Background scan through the runner: `scan --background --notify json`; `{"status": "skipped", "reason": "busy"}` (exit 0) posts nothing and leaves the scan due for the next tick
- [ ] Notifications, only from scheduled scans:
  - text from the decision: the amount and at most three groups, or the volume that is almost full (artboard 2); never an offer to clean directly;
  - actions Review (opens the Review window) and Remind me tomorrow;
  - with "Only notify during working hours" on, a decision made outside them is kept as pending in `UserDefaults` and posted at the first hourly tick inside them (`--scheduled` posts it without scanning); a newer decision replaces it;
  - after posting, `notify ack --bytes <n>`; not posted, no ack
- [ ] Notification permission: asked for at the first normal-mode launch after the first scan, or in the migration flow; never by a `--scheduled` instance. A `--scheduled` instance that finds it not yet determined sets a flag; the next popover open surfaces it
- [ ] Remind me tomorrow: `notify snooze --until <the notifying scan's slot + 24 hours>`. The first scheduled scan after it notifies whenever it is over the threshold or low on space (story "CLI: shared settings, types and the notify decision"). With a weekly schedule the due function adds one reminder scan at the first tick after the snooze ends. If the app was quit, macOS launches it to deliver the action: it starts in normal mode, keeps its status item, and the badge clears
- [ ] Attention badge on the status item until the Review window opens
- [ ] Migration from 0.1.0, offered and never silent:
  - on first launch, the popover offers to replace the old background scan; Settings → Permissions keeps "Old background scan found · Replace it" while `~/Library/LaunchAgents/com.smartandpoint.deepdrivecleaner.plist` exists;
  - Replace it: the schedule becomes 11:00 with the old interval, daily or weekly on Mondays (the 0.1.0 agent runs at 11:30, `crates/ddc-cli/src/launchd.rs`, and the app schedules whole hours); the old threshold goes into `settings.json`; then `deepdrivecleaner schedule uninstall`, and the app's agent is registered;
  - the migration flow also asks for notification permission

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Unit test: the due function for daily and weekly schedules, before, at and after the slot, and for a slot missed while asleep; a manual scan (a newer `last-scan.json`, no new scheduled-scan time) changes neither the next slot nor whether today's scheduled scan runs and may notify
- [ ] Unit test: with a weekly schedule, Remind me tomorrow makes the first tick after the snooze ends due (one reminder scan), and the regular weekly slot stays where it was
- [ ] Unit test: the notification text built from a recorded threshold decision names the amount and at most three groups; from a low-disk decision, the volume
- [ ] Unit test: a posted notification calls `notify ack` with the decision's amount; one held back by denied permission calls nothing
- [ ] Unit test: a decision at 22:00 with working hours on is not posted, is kept as pending, and is posted at the 09:00 tick of the next working day and acknowledged then; a newer decision before 09:00 replaces it
- [ ] Unit test: a `--scheduled` run that finds notification permission not determined requests nothing, sets the flag, and the next popover open shows the request; a normal-mode launch after the first scan asks once
- [ ] Unit test: Remind me tomorrow calls `notify snooze` with the start of tomorrow's slot for a daily schedule at 10:00, also when the user taps it hours after the notification
- [ ] Unit test with a fake `SMAppService`: every launch with scheduled scans on calls `register()`; `.requiresApproval` leaves the record unchanged; a later launch that finds `.enabled` writes it with the app's path; a failing `register()` clears it; turning scheduled scans off unregisters and clears it
- [ ] Test with a fake CLI: a background run answered with `{"status": "skipped", "reason": "busy"}` posts nothing, and the due function still says due
- [ ] Test with a temp home holding a 0.1.0 LaunchAgent plist, a fake CLI and a fake `SMAppService`: Replace it sets the schedule to 11:00 daily for a daily plist and 11:00 on Mondays for a weekly one, writes the threshold, calls `schedule uninstall` and registers the agent; declining changes nothing and the offer stays in Settings
- [ ] Manual check on the reference machine with the app quit: at the next tick the app is opened with `--scheduled`, scans, posts and quits; Review opens the Review window; Remind me tomorrow posts nothing until the first scheduled scan after the snooze, which posts if still over the threshold
- [ ] Manual check: after the first launch and the migration, System Settings → Login Items & Extensions lists the scheduled scan under "DeepDriveCleaner" with the app's icon, not under a developer name
- [ ] Manual check after the migration: the 0.1.0 LaunchAgent is gone; `launchctl print gui/$UID/com.smartandpoint.deepdrivecleaner` finds no job and `~/Library/LaunchAgents/com.smartandpoint.deepdrivecleaner.plist` does not exist
- [ ] Manual check: after the app is quit and replaced (as `brew upgrade` does), the next tick opens the new app and the scheduled scan runs
- [ ] Manual check: double-clicking the app in Finder during a scheduled scan shows the menu bar icon, and the app keeps running after the scan
- [ ] Manual check: with the app quit, Remind me tomorrow on a notification launches it in normal mode with its status item, and the badge is cleared
- [ ] Manual check: `ddc schedule install` refuses while the app's agent is registered, and installs after the app is removed

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- Why Login Items changes: the 0.1.0 agent's program is a bare binary outside any bundle, so System Settings lists it under the signer, "Evgenii Konev · 1 item", with a generic icon, and the background-activity banner says "Software from “Evgenii Konev”". An agent registered with `SMAppService.agent` from the bundle is attributed to the bundle. For any plist outside the bundle, `AssociatedBundleIdentifiers` (`man launchd.plist`) associates the job with the app; `ddc schedule install` sets it (story "CLI: shared settings, types and the notify decision").
- The developer name in Login Items and in the background-activity banner stays "Evgenii Konev" in 0.2.0; it becomes the organization's only after "Signing team: parameterise TEAM_ID, then switch to SmartAndPoint".
- Login Items lists two entries for the app: the login item and the scheduler agent.
- The notify rule and its state file stay in Rust (`notify.rs`); the app presents the decision and asks the CLI to record it.
- The last scheduled scan's time is the app's own record: `last-scan.json` also changes on a manual scan or a terminal `ddc scan`, which must not move the schedule.

## Dependencies

- [[epics/DDC-002/stories/story-app-skeleton-and-build-pipeline|App skeleton and build pipeline]]
- [[epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision|CLI: shared settings, types and the notify decision]]
- [[epics/DDC-002/stories/story-menu-bar-popover|Menu bar popover]]
- [[epics/DDC-002/stories/story-settings-folders-types-notifications|Settings: folders, types, notifications]]

## Attachments

- Design canvas: https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3

---

*Last updated: 2026-10-18*

---
type: story
id: "story-background-scan-and-notifications-in-the-app"
epic: "DDC-002"
title: "Background scan and notifications in the app"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-30
external_refs: {design_canvas: "https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV"}
tags: [app, notifications, schedule]
code_refs: ["README.md", "scripts/build-app.sh", "app/DeepDriveCleaner.xcodeproj/project.pbxproj", "app/DeepDriveCleaner/AppDelegate.swift", "app/DeepDriveCleaner/DebugMenu.swift", "app/DeepDriveCleaner/PermissionsSettingsView.swift", "app/DeepDriveCleaner/PopoverView.swift", "app/DeepDriveCleaner/SystemAgentService.swift", "app/DeepDriveCleaner/SystemNotifications.swift", "app/DeepDriveCleaner/main.swift", "app/LaunchAgents/com.smartandpoint.deepdrivecleaner.scheduler.plist", "app/LaunchAgents/com.smartandpoint.deepdrivecleaner.scheduler-open.plist", "app/Packages/DDCKit/Sources/DDCKit/AppPreferences.swift", "app/Packages/DDCKit/Sources/DDCKit/CLIRunner.swift", "app/Packages/DDCKit/Sources/DDCKit/Formatting.swift", "app/Packages/DDCKit/Sources/DDCKit/LaunchMode.swift", "app/Packages/DDCKit/Sources/DDCKit/LegacyAgent.swift", "app/Packages/DDCKit/Sources/DDCKit/NotificationText.swift", "app/Packages/DDCKit/Sources/DDCKit/NotifyReport.swift", "app/Packages/DDCKit/Sources/DDCKit/PopoverModel.swift", "app/Packages/DDCKit/Sources/DDCKit/ScanDue.swift", "app/Packages/DDCKit/Sources/DDCKit/Schedule.swift", "app/Packages/DDCKit/Sources/DDCKit/ScheduledWork.swift", "app/Packages/DDCKit/Sources/DDCKit/SchedulerAgent.swift", "app/Packages/DDCKit/Sources/DDCKit/SettingsModel.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ScheduledScanTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ContractTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/PopoverSupport.swift"]
specs: []
started_at: "2026-10-28T03:23:01.141Z"
closed_at: "2026-10-30T06:45:41.994Z"
plan_updated_at: "2026-10-28T03:23:01.141Z"
---

# Background scan and notifications in the app

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
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

- [x] Scheduler agent, self-healing (ADR §5):
  - at every launch with scheduled scans on, call `register()` for `com.smartandpoint.deepdrivecleaner.scheduler` (idempotent), which also recovers an agent approved later or dropped when `brew` replaced the bundle;
  - write the `scheduler` record to `settings.json` whenever the status is `.enabled`; clear it only when scheduled scans are off (and unregister) or registration fails; `.requiresApproval` leaves it as it is;
  - the Settings state follows `SMAppService.status`
- [x] Due function, shared with the popover and Settings:
  - the schedule (daily or weekly, a whole hour) and the time of the last **scheduled** scan, kept in `UserDefaults`, not `last-scan.json`;
  - a slot missed while the Mac slept is due at the next tick;
  - with no scheduled scan recorded yet (a new install), the first tick is due; the clean-machine check of "Distribution: the 0.2.0 cask with the app" relies on it;
  - a Scan now or a terminal `ddc scan` neither moves the slot nor suppresses that day's notification;
  - a pending reminder scan (Remind me tomorrow with a weekly schedule) is due at the first tick after the snooze ends
- [x] `--scheduled` launch: no status item; scan if due, post a pending notification if one is waiting, quit. A reopen event while running normally: scan if due, and bring windows forward only when the app is active. Promotion on a user launch during a scheduled scan is already built (`LaunchMode`)
- [x] Background scan through the runner: `scan --background --notify json`; `{"status": "skipped", "reason": "busy"}` (exit 0) posts nothing and leaves the scan due for the next tick
- [x] Read the preferences that "Settings: folders, types, notifications" stores (2026-10-28):
  - the `Schedule*` keys (`ScheduleEnabled`, on by default, daily at 10:00; `ScheduleRepeat`, `ScheduleWeekday`, `ScheduleHour`);
  - `AfterScheduledScan`: `notify`, or `menuBarOnly`, which posts nothing and only sets the badge (subject to `ShowMenuBarBadge`);
  - `NotifyWorkingHoursOnly`.

  Fill that story's seams: `nextScan`, `SchedulerControl`, and the Permissions rows.
- [x] Notifications, only from scheduled scans:
  - text from the decision: the amount and at most three groups, or the volume that is almost full (artboard 2); never an offer to clean directly;
  - actions Review (opens the Review window) and Remind me tomorrow;
  - with "Only notify during working hours" on, a decision made outside them is kept as pending in `UserDefaults` and posted at the first hourly tick inside them (`--scheduled` posts it without scanning); a newer decision replaces it;
  - after posting, `notify ack --bytes <n>`; not posted, no ack
- [x] Notification permission: asked for at the first normal-mode launch after the first scan, or in the migration flow; never by a `--scheduled` instance. A `--scheduled` instance that finds it not yet determined sets a flag; the next popover open surfaces it. The flag, and the offer to replace the 0.1.0 agent, show through the popover's notice slot ("Menu bar popover").
- [x] Remind me tomorrow: `notify snooze --until <the schedule hour on the day after the tap>` (B7). The first scheduled scan after it notifies whenever it is over the threshold or low on space (story "CLI: shared settings, types and the notify decision"). With a weekly schedule the due function adds one reminder scan at the first tick after the snooze ends. If the app was quit, macOS launches it to deliver the action: it starts in normal mode, keeps its status item, and the badge clears
- [x] A posted notification sets the attention badge built in "Menu bar popover" (`notificationPosted()`); it clears when the Review window opens
- [x] Pause from "Menu bar popover" (`ScansPaused`): the due function reports nothing due while paused, so a scheduled launch or a reopen scans nothing, and the footer shows "Scans paused" instead of a next scan. After Resume a missed slot is due at the next tick (the agent's hourly launch or a reopen), not at Resume, and until then the footer shows "Next scan within the hour", never a slot in the past. A pending working-hours notification still posts while paused: it reports a scan already made
- [x] Migration from 0.1.0, offered and never silent:
  - while `~/Library/LaunchAgents/com.smartandpoint.deepdrivecleaner.plist` exists, the popover offers to replace the old background scan until the offer is dismissed, and Settings → Permissions keeps "Old background scan found · Replace It". This covers agents from 0.1.0 and from a 0.2.x `ddc schedule install`;
  - while that plist exists, the app neither registers its agent nor runs scheduled scans (B1);
  - Replace it: the schedule becomes 11:00 with the old interval, daily or weekly on Mondays (the 0.1.0 agent runs at 11:30, `crates/ddc-cli/src/launchd.rs`, and the app schedules whole hours); the old threshold goes into `settings.json`; then `deepdrivecleaner schedule uninstall`, and the app's agent is registered;
  - the migration flow also asks for notification permission

## Implementation Plan

Planned on 2026-10-28 with `projectstore:planner`. No spec covers this story. No Rust change is needed: `notify ack|snooze`, `scan --background --notify json`, the `scheduler` record and the `schedule install` refusal shipped with "CLI: shared settings, types and the notify decision".

**Decisions taken at the gate.** The owner delegated these on 2026-10-25 and 2026-10-28.
- **B1.** While the legacy plist exists, the app neither registers its agent nor runs scheduled scans: the due function gets a `legacyAgentFound` input and reports nothing due. The footer and the Schedule pane say "Replace the old background scan first". Two schedulers therefore never run together. The CLI's own agent keeps notifying through osascript until the user replaces it, so declining is not a dead end.
- **B2.** The migration offer shows whenever the legacy plist exists and the offer has not been dismissed, not only at the first launch. 0.2.x users have launched the app already.
- **B3.** `AfterScheduledScan == menuBarOnly` posts nothing and sends no ack. It sets the badge only, subject to `ShowMenuBarBadge`.
- **B4.** A `--scheduled` instance that finds notification permission not determined sets the flag, drops the notification and sends no ack. It never asks for permission.
- **B5.** `ScheduleWeekday` uses `Calendar` weekday numbering (Sunday = 1). launchd's Weekday 1 (Monday) maps to 2.
- **B7.** Remind Me Tomorrow is anchored on the tap, not on the scan. `until` is `ScheduleHour` on the calendar day after the tap, and so is `ReminderScanAt`. A notification held back by working hours, or one tapped days later in Notification Center, then still means "tomorrow".
- **B8.** Denied notification permission: the decision is dropped and no ack is sent, but the badge is set, so the icon still shows that something can be freed.
- **B9.** Turning scheduled scans off clears `ReminderScanAt` and `PendingNotification`. A 0.2.x user who removed the CLI agent gets the app's daily scan at the first launch of 0.3.0, as Settings shows it (D1). The release notes say so and point to Settings → Schedule.
- **B6.** The bundled plist uses `BundleProgram Contents/MacOS/DeepDriveCleaner` with `ProgramArguments ["DeepDriveCleaner", "--tick"]` (argv[0] included), the ADR's fallback.
  - `main.swift`'s `--tick` opens the app by its own bundle path: `open -gj <Bundle.main.bundleURL> --args --scheduled`, not `-b <id>`. A Debug copy and a release copy therefore never launch each other.
  - **Reasoning, not yet observed.** SMAppService documents bundle programs, and Login Items should then attribute the agent to the bundle. No registration of this chain has been observed yet.
  - **The first manual check of the release settles it.** Register the agent from the release build, confirm that a `--scheduled` launch appears in `app.log` within the hour, and check the Login Items attribution.
  - **If the bundle program is refused,** the app falls back at runtime, so the release does not wait on the check. The bundle carries a second plist, `com.smartandpoint.deepdrivecleaner.scheduler-open.plist`, with the `/usr/bin/open -gjb <id> --args --scheduled` program and its own label. When the primary's status after `register()` is neither `.enabled` nor `.requiresApproval`, `SchedulerAgent` unregisters the primary and registers the fallback.
  - If both are refused, the popover's notice names Login Items and `ddc schedule install`. This matters most inside Replace It, where the user has just lost the old agent.

**Route.**
1. **Runner kinds.** `CLIRunner` gains `.backgroundScan`, `.notify` and `.schedule`, because `.scan` coalesces and would hand one caller another's output. `FakeRunner` replies per kind. A `NotifyReport` decodes the background scan's decision, with contract tests against `notify.rs` and `docs/notify.schema.json`.
2. **The due function.** `ScanDue.swift` is pure and takes the schedule, the last scheduled scan, the reminder, paused, now and the calendar.
   - Inputs also include `legacyAgentFound`, from `SchedulerControl`.
   - `due` returns `.slot` or `.reminder`, or nil when off, paused, or when the legacy agent is found. A new install's first tick is due, and a slot missed while asleep is due.
   - `next` gives nil when off or when the legacy agent is found, a date ≤ now when due, else the earlier of the next slot and the reminder. It feeds the `nextScan` seam of "Settings: folders, types, notifications" and the popover footer.
   - `Wording.nextScan` (the footer) gains the branch "within the hour" for a date ≤ now. The seam returns an enum (`.off`, `.replaceOldScanFirst`, `.at(Date)`) instead of `Date?`, and the popover footer and `Schedule.nextScanLine` both read it, so "Replace the old background scan first" has its own case.
   - New `AppPreferences` keys: `LastScheduledScan` (set only after `scanned`), `ReminderScanAt`, `PendingNotification`, `NotificationPermissionPending`, `NotificationPermissionAsked` and `OldScanOfferDismissed`.
3. **Notification text.** `NotificationText.swift`, from artboard 2:
   - threshold: the amount and at most three top groups;
   - low disk: the volume and its free space;
   - never an offer to clean.
4. **One tick.** `ScheduledWork.swift` runs it:
   - when due, `PopoverModel.scheduledScan()`, which waits for a user scan and never joins it, runs `scan --background --notify json`, then reloads the report;
   - busy: nothing changes;
   - scanned: record the scan, clear `ReminderScanAt`, and let the decision replace the pending one;
   - then flush the pending decision, in this order: `menuBarOnly` (B3); working hours, Monday to Friday from 9:00 inclusive to 19:00 exclusive, keep it pending; authorization:
     - authorized: post, `notify ack --bytes`, set the badge;
     - denied: drop it, send no ack, set the badge (B8);
     - not determined: B4.
   - A scheduled run never activates the app.
   - The post's completion and the `notify ack` are awaited before `.workFinished`, so a `--scheduled` instance quits only after both.
   - Scan Now pressed during a scheduled scan waits for it, then runs a foreground scan. The popover shows the scheduled scan as scanning, with elapsed time only.
5. **Notification responses.**
   - Review loads the report first, then opens the Review window.
   - Remind Me Tomorrow runs `notify snooze --until`, set to the schedule hour on the day after the tap (B7), in RFC 3339 UTC with `Z`, and clears the badge. With a weekly schedule it also sets `ReminderScanAt` to the same time.
6. **The scheduler agent.** `SchedulerAgent.swift` sits behind an `AgentService` port and conforms to Settings' `SchedulerControl`.
   - At every launch with scheduled scans on and no legacy plist, it calls `register()`.
   - It decides by the status after the call, not by whether the call threw, and tries the fallback plist when the primary is refused (B6): `.enabled` writes `scheduler {by: app, app: bundle path}` through `SettingsStore`, only when the record changes; `.requiresApproval` leaves the record; any other status clears it.
   - Turning scans off unregisters the agent and clears the record.
7. **Migration.** `LegacyAgent.swift` reads the plist into memory first, because the uninstall deletes it. It reads with `PropertyListSerialization` and gets the interval, hour, weekday and an optional `--threshold`. Contract tests pin `LABEL`, 11 and Weekday 1 in `launchd.rs`. Replace It runs these steps in order:
   1. run `schedule uninstall`; anything but `.done` stops the flow, changes nothing and keeps the offer;
   2. write the schedule: 11:00, daily or Monday (B5);
   3. write the threshold through `SettingsStore`;
   4. register the agent;
   5. ask for notification permission.

   Declining sets only `OldScanOfferDismissed`.
8. **Popover and Settings.**
   - `Notice` gains actions: Allow Notifications and Replace It.
   - `refresh()` raises them from the pending-permission flag and from the presence of the legacy plist.
   - Settings' Permissions rows: Scheduled scan, with Open Login Items; Old background scan found · Replace It.
   - The hourly reopen brings windows forward only when `NSApp.isActive`.
9. **App target.**
   - `SystemAgentService.swift` wraps `SMAppService.agent(plistName:)`.
   - `SystemNotifications.swift` wraps `UNUserNotificationCenter`: the `scan-result` category with the actions Review (foreground) and Remind Me Tomorrow, one fixed request id, and a body click handled as Review.
   - The plist changes per B6.
   - `AppDelegate` wiring: the category is registered in `applicationWillFinishLaunching`; at every launch the agent heals itself; a normal launch asks for notification permission once; `.startScheduledWork` and `.scanIfDue` run `ScheduledWork`.
   - DDCKit never imports `UserNotifications` or `ServiceManagement`, and a contract test checks that.
10. **Docs and checks.** README; `make check`, `make app-test`; an unsigned build; `projectstore:reviewer`.

**Pitfalls:**
- A tick that arrives during a scheduled scan longer than an hour reaches the running instance as a reopen and promotes it, so the status item appears unasked. This is accepted: a scan over an hour is rare, and the result is visible, not harmful.
- The runner coalescing `.scan`.
- `UNUserNotificationCenter.current()` crashes outside a bundle, so it never runs in tests.
- Debug and release builds share a bundle id: unregister the Debug build's agent before dogfooding a release.
- Never pass `--threshold` from the app, so that `settings.json` wins.
- Never run a real `schedule uninstall`, `notify ack` or `notify snooze` from tests.
- Inject the calendar and time zone in tests.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [x] Unit test: the due function for daily and weekly schedules, before, at and after the slot, and for a slot missed while asleep; a manual scan (a newer `last-scan.json`, no new scheduled-scan time) changes neither the next slot nor whether today's scheduled scan runs and may notify; the popover footer shows this function's next slot through the schedule seam of "Menu bar popover" — evidence: `ScheduledScanTests.swift` `dueFunction`, `manualScanKeepsTheSlot` (commit dad390e)
- [x] Unit test: with a weekly schedule, Remind me tomorrow makes the first tick after the snooze ends due (one reminder scan), and the regular weekly slot stays where it was — evidence: `ScheduledScanTests.swift` `weeklyReminder`, `catchUpKeepsReminder` (commit dad390e)
- [x] Unit test: the notification text built from a recorded threshold decision names the amount and at most three groups; from a low-disk decision, the volume — evidence: `ScheduledScanTests.swift` `notificationText` (commit dad390e)
- [x] Unit test: a posted notification calls `notify ack` with the decision's amount; one held back by denied permission calls nothing — evidence: `ScheduledScanTests.swift` `ackAndDenied` (commit dad390e)
- [x] Unit test: a decision at 22:00 with working hours on is not posted, is kept as pending, and is posted at the 09:00 tick of the next working day and acknowledged then; a newer decision before 09:00 replaces it — evidence: `ScheduledScanTests.swift` `workingHours`, `pendingPostsWhilePaused` (commit dad390e)
- [x] Unit test: a `--scheduled` run that finds notification permission not determined requests nothing, sets the flag, drops the decision and sends no ack; the next popover open shows the request; a normal-mode launch after the first scan asks once — evidence: `ScheduledScanTests.swift` `permissionNotDetermined` (commit dad390e)
- [x] Unit test: with permission denied, the decision is dropped with no ack and the badge is set (B8) — evidence: `ScheduledScanTests.swift` `ackAndDenied` (commit dad390e)
- [x] Unit test: the footer wording for a due date at or before now reads "within the hour"; while the legacy plist exists, nothing is due and the footer says to replace the old background scan — evidence: `ScheduledScanTests.swift` `footerWording` (commit dad390e)
- [x] Unit test: turning scheduled scans off clears the reminder and the pending notification (B9) — evidence: `ScheduledScanTests.swift` `offClears` (commit dad390e)
- [x] Unit test: Remind Me Tomorrow anchors on the tap (B7) — evidence: `ScheduledScanTests.swift` `remindAnchorsOnTheTap` (commit dad390e):
  - a daily 10:00 schedule tapped at 10:05 snoozes until 10:00 the next day;
  - a notification held by working hours and posted Monday 09:00, tapped at 09:05, snoozes until Tuesday 10:00;
  - a tap three days after posting snoozes until 10:00 on the day after the tap.
- [x] Unit test with a fake `SMAppService`: every launch with scheduled scans on calls `register()`; `.requiresApproval` leaves the record unchanged; a later launch that finds `.enabled` writes it with the app's path; a failing `register()` clears it; turning scheduled scans off unregisters and clears it — evidence: `ScheduledScanTests.swift` `agentHeals` (commit dad390e)
- [x] Test with a fake CLI: a background run answered with `{"status": "skipped", "reason": "busy"}` posts nothing, and the due function still says due — evidence: `ScheduledScanTests.swift` `busy` (commit dad390e)
- [x] Test with a temp home holding a 0.1.0 LaunchAgent plist, a fake CLI and a fake `SMAppService` — evidence: `ScheduledScanTests.swift` `replaceIt`, `migrationRefusals`, `legacyRead` (commit dad390e):
  - Replace It sets the schedule to 11:00 daily for a daily plist, and to 11:00 on Mondays for a weekly one;
  - it writes the threshold, calls `schedule uninstall` and registers the agent;
  - a failed uninstall changes nothing (no schedule, threshold or registration) and keeps the offer;
  - declining changes nothing but hides the popover offer, and the offer stays in Settings;
  - while the plist exists, no launch registers the app's agent, and a reopen scans nothing (B1);
  - the fake uninstall removes the plist, and the schedule and threshold still come from what it held.
- [x] Unit test with a fake `SMAppService` — evidence: `ScheduledScanTests.swift` `fallback` (commit dad390e):
  - a refused primary plist leads to registering the fallback plist;
  - when both are refused, the record is cleared and the notice names Login Items and `ddc schedule install` (B6).
- [x] Unit test: the tick's arguments are `["-gj", <bundle path>, "--args", "--scheduled"]`, from a pure DDCKit function that `main.swift` uses — evidence: `ScheduledScanTests.swift` `tickArguments` (commit dad390e)
- [x] Unit test: with "Only show it in the menu bar", a scheduled scan over the threshold posts nothing, sends no ack and sets the badge (B3) — evidence: `ScheduledScanTests.swift` `menuBarOnly` (commit dad390e)
- [x] Unit test: the bundled scheduler plist is `BundleProgram Contents/MacOS/DeepDriveCleaner` with `ProgramArguments ["DeepDriveCleaner", "--tick"]` (B6), and its label is the agent label the app registers — evidence: `ScheduledScanTests.swift` `bundledPlists` (commit dad390e)
- [x] Contract test: no DDCKit source imports `UserNotifications` or `ServiceManagement` — evidence: `ContractTests.swift` `noSystemFrameworks` (commit dad390e)
- Manual checks (8) moved to [Owner checks: manual acceptance across stories](../../../ops/owner-checks-manual-acceptance-across-stories.md) on 2026-10-30, by the owner's decision.

## Final Summary

**What changed** (code commit dad390e, released in 0.3.0 with 9d0418c; 118a6a9 adds the fallback scheduler agent to the cask's `zap`):
- The app runs the scheduled scan itself: a bundled scheduler agent registered through `SMAppService`, a `--tick` that opens the app with `--scheduled`, and a fallback plist when the bundle program is refused (B6).
- A pure due function (`ScanDue.swift`) for daily and weekly slots, reminders and slots missed while asleep. The popover footer and Settings read it.
- Notifications with Review and Remind me tomorrow, working hours, "Only show it in the menu bar", and the denied and not-determined permission paths. The decision and its state stay in the CLI (`notify ack`, `notify snooze`).
- The offered migration from the 0.1.0 LaunchAgent (Replace It). While the old plist exists, the app registers no agent and runs no scheduled scan (B1).

**Tests run:** `ScheduledScanTests.swift` (due function, reminders, notification text, ack and denied permission, working hours, permission not determined, footer wording, agent healing, busy, migration, fallback, tick arguments, menu bar only, bundled plists) and `ContractTests.noSystemFrameworks`, all at commit dad390e.

**Remaining risk:** the eight manual checks moved to the owner list: the real tick and `--scheduled` launch, the Login Items attribution, the 0.1.0 agent gone after the migration, a tick after an in-place upgrade, Finder and Dock behaviour during a scheduled scan, Remind me tomorrow with the app quit, and the `ddc schedule install` refusal against the real agent. The plan recorded that the bundle-program registration had not been observed; the first of these checks settles it.

## Technical Notes

- Why Login Items changes: the 0.1.0 agent's program is a bare binary outside any bundle, so System Settings lists it under the signer, "Evgenii Konev · 1 item", with a generic icon, and the background-activity banner says "Software from “Evgenii Konev”". An agent registered with `SMAppService.agent` from the bundle is attributed to the bundle. For any plist outside the bundle, `AssociatedBundleIdentifiers` (`man launchd.plist`) associates the job with the app; `ddc schedule install` sets it (story "CLI: shared settings, types and the notify decision").
- The developer name in Login Items and in the background-activity banner stays "Evgenii Konev" in 0.3.0; it becomes the organization's only after a switch to an organization team.
- Login Items lists two entries for the app: the login item and the scheduler agent.
- The notify rule and its state file stay in Rust (`notify.rs`); the app presents the decision and asks the CLI to record it.
- From "CLI: shared settings, types and the notify decision":
  - `notify snooze --until` takes RFC 3339 UTC with `Z` only (`2026-10-20T09:00:00Z`); anything else exits 2.
  - `scan --background --notify json` exits 1 with nothing on stdout when `settings.json` is malformed or none of its `roots` exists: the app shows the error and posts nothing.
  - `volume`, `free` and `total` in the decision name the volume under 10 % free also while a snooze holds the notification back; the repeat rule never holds back low disk, so an almost full disk notifies at every scheduled scan until cleaned or snoozed; the app reads them only when `notify` is true.
- The last scheduled scan's time is the app's own record: `last-scan.json` also changes on a manual scan or a terminal `ddc scan`, which must not move the schedule.

## Dependencies

- [[epics/DDC-002/stories/story-app-skeleton-and-build-pipeline|App skeleton and build pipeline]]
- [[epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision|CLI: shared settings, types and the notify decision]]
- [[epics/DDC-002/stories/story-menu-bar-popover|Menu bar popover]]
- [[epics/DDC-002/stories/story-settings-folders-types-notifications|Settings: folders, types, notifications]]

## Attachments

- Design canvas: https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV

---

*Last updated: 2026-10-30*

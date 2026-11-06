---
type: story
id: "story-scheduled-scans-register-the-agent-again-after-an-update"
epic: "DDC-002"
title: "Scheduled scans: register the agent again after an update"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-04
updated: 2026-11-06
external_refs: {}
tags: [app, scheduler, updates, bug]
code_refs:
  - app/DeepDriveCleaner/AppDelegate.swift
  - app/DeepDriveCleaner/PopoverView.swift
  - app/DeepDriveCleaner/ScheduleSettingsView.swift
  - app/DeepDriveCleaner/SettingsWindowController.swift
  - app/Packages/DDCKit/Sources/DDCKit/AgentTicks.swift
  - app/Packages/DDCKit/Sources/DDCKit/AppPreferences.swift
  - app/Packages/DDCKit/Sources/DDCKit/Formatting.swift
  - app/Packages/DDCKit/Sources/DDCKit/PopoverModel.swift
  - app/Packages/DDCKit/Sources/DDCKit/ScanDue.swift
  - app/Packages/DDCKit/Sources/DDCKit/Schedule.swift
  - app/Packages/DDCKit/Sources/DDCKit/ScheduledWork.swift
  - app/Packages/DDCKit/Sources/DDCKit/SchedulerAgent.swift
  - app/Packages/DDCKit/Sources/DDCKit/SettingsModel.swift
  - app/Packages/DDCKit/Tests/DDCKitTests/ContractTests.swift
  - app/Packages/DDCKit/Tests/DDCKitTests/ScheduledScanTests.swift
  - app/Packages/DDCKit/Tests/DDCKitTests/SettingsTests.swift
specs: []
started_at: "2026-11-06T10:44:20.786Z"
closed_at: "2026-11-06T10:44:20.968Z"
plan_updated_at: "2026-11-06T10:44:20.786Z"
---

# Scheduled scans: register the agent again after an update

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | done |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner, 2026-11-04: the popover says "Next scan within the hour", but no scheduled scan has run on their Mac since 2026-10-31. The app log's last `tick:` line is from 0.11.1; every launch since, from 0.12.1 to 0.34.0, logs "scheduler: com.smartandpoint.deepdrivecleaner.scheduler.plist enabled". `launchctl print` of the agent says `job state = spawn failed`, `last exit code = 78: EX_CONFIG`, `properties = … resolve program`, and `parent bundle version = 9`, while the installed app is build 96. So launchd keeps the registration of an old build and cannot start the bundle program of the new one, and `SMAppService.status` still answers `.enabled`, so `SchedulerAgent.heal()`, whose `register` is idempotent, changes nothing. Each hourly tick fails before the app runs, and nothing tells the person.

What it breaks: no scheduled scan and no notification; no `ddc space` check on the tick; the saved report stays as old as the last scan by hand (0.29.1 on that Mac), so after each report schema change the popover says "No scan yet" (story "Scan after an update when the saved report cannot be read"), and "Grew since …" (story "ddc changes: what grew between the last two scans", DDC-001) has no pair to compare.

The cause is known only as far as `launchctl` shows it: when and why the registration stopped following the bundle is not known (a Sparkle update replaces the bundle in place; build 9 is far older than 0.11.1, which still ticked). The plan gate starts with Apple's documentation and known reports on `SMAppService` agents after an app update.

What the app should do:

- **Register again after an update in place.** `heal()` records the bundle path and the build it registered (`AppPreferences` key `SchedulerRegistration`, type `RegisteredBundle` {path, build}, since `SchedulerRegistration` already names the allowed/approval/off state; written only while scheduled scans are on; scheduled scans off leave it as it is). At a launch from the **same path** with **another build** (an update in place, by Sparkle or Homebrew), it unregisters the agent and registers it again, then records both. A launch from **another path** (a copy elsewhere) does not re-register: it logs it and keeps today's idempotent `register`. **A Debug build never re-registers and records nothing, Fix included** (`AppDelegate` passes `reregisters: false` under `#if DEBUG`): Debug and Release share the defaults and the build number, and with no record yet two Debug launches in a row would otherwise take the agent onto a DerivedData bundle (the planner, 2026-11-06). Same path, same build: as today. **With no record yet** (the first launch of the build that ships this, the owner's case), the path comes from `settings.json`'s `scheduler.app`, which `heal()` writes while the agent is `.enabled`: the same path re-registers and records; another path only logs and registers as today; no `scheduler.app` yet (a first registration) records without re-registering. When the re-registration ends in `.requiresApproval` or the unregister throws, the record is still written (no loop at every launch) and Settings shows the state as today.
- **Say when the agent's ticks do not arrive.** The tick records when an **agent's** tick ran (`LastAgentTickAt`): a `--scheduled` launch, or a reopen while the app runs (the agent's `open` arrives as a reopen; a click on the Dock icon of a menu bar app is rare noise). The launch tick of the story "Scan after an update when the saved report cannot be read" does not record it. With scheduled scans on and the agent `.enabled` (for `.requiresApproval`, the line says "Scheduled scans wait for approval in Login Items" at once, and leads to Settings → Permissions), when two hourly ticks are missing, counted in awake time: an agent tick records `ProcessInfo.systemUptime` beside `LastAgentTickAt` (on macOS it does not advance in sleep); "not running" is `systemUptime − tickUptime > 2 h`; when the saved uptime is larger than the current one, the Mac restarted since, and the rule is "no agent tick since boot and `systemUptime > 2 h`". So neither a reboot nor a long sleep counts, the popover's next-scan line and Settings → Schedule say "Scheduled scans are not running" with a **Fix** action that runs the re-registration. If ticks still do not arrive after Fix, the line offers the existing `SchedulerAgent.refused` route (Login Items, or `ddc schedule install`).
- The `--scheduled` instance is launched by LaunchServices (`open`), not the agent's job (`--tick` exits after `open`), so its own unregister should not end it. The real risk is a crash between unregister and register: no ticks until the next launch, which re-registers because the record was not written yet.
- **"Not running" needs a baseline** (the planner, 2026-11-06): with no `LastAgentTickAt` yet, "no agent tick since boot and uptime > 2 h" would fire at the first launch of the build that ships this on any Mac up more than 2 h. So a baseline {time, uptime} is written once, when there is neither a tick record nor a baseline (a later launch without a tick does not move it), and again at a re-registration and at Fix, and "not running" is measured from the later of the last agent tick and that baseline. The reboot rule only catches a restart while the new uptime is below the saved one. Built in the second pull request, so it errs late rather than with a false alarm: the start is also written when the registration becomes allowed (scans turned on again, Replace It) and kept fresh while macOS waits for approval; the app's own process start counts too, so after a long logout the line waits two hours of the new session (the signal comes two hours after each login at the earliest). After Fix with still no tick, the line leads to Settings → Schedule, which names Login Items and `ddc schedule install`; while macOS waits for approval it says so at once and leads to Permissions.

## Decomposition

- [x] Research: `SMAppService` agents after an app update: what launchd keeps (a bundle path, a version, the signature), why `status` says `.enabled` while the job cannot spawn, and whether `unregister()` + `register()` keeps the approval and shows no "Background Items Added" notice. Start points: build 9 is 0.4.1 (`git log -S "CURRENT_PROJECT_VERSION = 9;"`); 0.11.1 (build 17) still ticked and 0.12.1 (build 19) did not, and between the two only the version changed in `app/`, `Makefile`, `scripts/` and `packaging/`, so the trigger is outside the code (how the update was installed, by Homebrew or Sparkle, per `app.log`; the macOS version; the signature)
- [x] `SchedulerAgent.heal()`: the {path, build} record and the re-registration after an update in place
- [x] `ScheduledWork`: `LastAgentTickAt` from agent ticks only; `NextScan`: a "not running" state from it and the uptime
- [x] `PopoverView` and Settings → Schedule: the line, Fix, and the refused route after Fix
- [x] Menu bar ADR §5 and the Sparkle ADR's line "heal() re-registers the scheduler agent at launch": a dated amendment each (idempotent `register` does not cover an update in place)
- [x] `registrationStep`: a `scheduler.app` whose bundle is gone re-registers (second pull request), with a test where both the record and `scheduler.app` point at a gone bundle
- [x] DDCKit tests with the fake `AgentService`

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

From the planner's pass (2026-11-06), no covering spec, in two pull requests. PR 1 (#129): `RegisteredBundle` in `AppPreferences` (key `SchedulerRegistration`); `SchedulerAgent.registrationStep` (pure) and the re-registration in `heal()`; `bundleVersion`, `reregisters` (false in Debug) and `bundleExists` seams; `healAtLaunch()` in the `change` queue. PR 2 (#130): `AgentTickMark`, `LastAgentTickAt` and `AgentWatchedSince`; `AgentTicks.stopped`; `ScheduledWork.tick(_ source:)`, `watchAgent()`, `next()` with `NextScan.notRunning`; `SchedulerAgent.fix()`; `SettingsDestination.approval`; the popover footer's and Settings → Schedule's Fix; `registrationStep(settingsAppExists:)`. ADR amendments, critic, reviewer at each step.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [x] Test (DDCKit, fake `AgentService`): a launch from the recorded path with another build, scheduled scans on, unregisters then registers the primary plist and records {path, build}; the same build only registers; another path does not re-register and logs it; with no record, a bundle path equal to `settings.json`'s `scheduler.app` re-registers and records, and another path does not; scheduled scans off unregister as today and record nothing; a re-registration ending in `.requiresApproval` or an unregister that throws still records, and the next launch of that build does not re-register; a record whose bundle is gone defers to `scheduler.app`, a `scheduler.app` whose bundle is gone re-registers, one that still exists elsewhere does not; a Debug build's Fix moves nothing — evidence: ScheduledScanTests `registrationStep`, `reregistersAfterAnUpdate`, `noRecordYet`, `otherPathAndOff`, `recordsWhateverTheStatus`, `reregistersTheFallback`, `movedApp`, `goneSettingsApp`, `launchHealQueues`, `fix` (a Debug build's Fix), `agentHeals`
- [x] Test (DDCKit): `NextScan` is "not running" when no agent tick ran for more than two hours of uptime with scheduled scans on and the agent enabled; an agent tick resets it, the launch tick does not; right after a launch with less than two hours of uptime it is not "not running"; a long sleep after the last tick does not count (uptime against uptime); after a restart, only awake time since boot counts; Fix runs the re-registration, and after a Fix with still no tick the line offers the refused route; with the agent at `.requiresApproval` the line says it waits for approval in Login Items and leads to Settings → Permissions; a later launch without a tick does not move the watch's start — evidence: ScheduledScanTests `agentTicksStopped`, `notRunning`, `waitsForApproval`, `fix`, `onAgainStartsTheWatch`, `joiningAndLogout`; SettingsTests `notRunningRow`; ContractTests `launchTickWaits` (only the launch tick is `.launch`)
- [x] `make ddckit-test` passes and the Gate's `app` job builds the app — evidence: `make ddckit-test` 552 passed (and the pre-push hook's runs); PRs #129 and #130 checks green, `app` among them; a Debug `xcodebuild` of the app target compiled before each push (the `app` job runs `make app-test`, not a build of the app target)
- [x] The two ADR amendments are written and the critic has read them — evidence: menu bar ADR §5 "Registration heals itself" (amended 2026-11-06, and the brew-upgrade caveat) and the Sparkle ADR's "Grants and agents"; critic approved in its third round
- [ ] Owner check (runbook "Owner checks: manual acceptance across stories"): after the update that ships this, `launchctl print` of the agent shows the new build and no "spawn failed", the app log has a `tick:` line within the hour, macOS showed no Login Items notice, and the agent is still allowed; `launchctl print` of the login item (`SMAppService.mainApp`) is checked the same way; and after a logout and login, or a restart, `launchctl print` still shows the new build with no "spawn failed" and a `tick:` arrives within the hour

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->


**What changed.** PR #129: `heal()` records the bundle path and build it registered the scheduler agent from (`RegisteredBundle`, defaults key `SchedulerRegistration`), and when the same path launches another build it unregisters what is registered and registers again. A copy elsewhere only registers; a Debug build never re-registers; with no record, `settings.json`'s `scheduler.app` stands for the path; a record or `scheduler.app` whose bundle is gone does not hold it back; the record is written whatever the status. The launch's heal runs in the same queue as Settings' on and off. PR #130: agent ticks record when they ran (clock and uptime); two hours of awake time with no agent tick make the popover footer and Settings → Schedule say "Scheduled scans are not running" with Fix (a forced re-registration); still none after Fix, Settings names Login Items and `ddc schedule install`; an agent waiting for approval says so at once and leads to Permissions. The watch never raises a false alarm at a first launch, after sleep, after scans were turned on again, Replace It or an approval, or after a long logout.

**Why.** The owner, 2026-11-04: no scheduled scan had run since 2026-10-31; launchd held build 9's registration and could not start the bundle program (`spawn failed`, exit 78) while `SMAppService` said enabled, and nothing told the person.

**Tests.** ScheduledScanTests (the decision table, an update in place, no record yet, another path, Debug, off, approval pending and a throwing unregister, the fallback plist, a moved app, a gone `scheduler.app`, the launch queue, the watch: sleep, restart, logout, a joining tick, not running, Fix, Fix with still no tick, approval, scans on again), SettingsTests `notRunningRow`, ContractTests `launchTickWaits`. DDCKit 552 passed; a Debug `xcodebuild` before each push; PRs #129 and #130 green.

**Risks and follow-ups.** The repair is expected, not proven: what broke the registration is unknown, and two developer reports say a pin survived unregister and register (they moved to a per-build label). The owner check, including a logout or restart, decides; plan B (re-register only on the "not running" signal) if macOS asks for approval at every update, plan C (a per-build label) if the repair does not survive a login. A Homebrew user with the app quit is repaired only when the app is opened. The signal comes two hours of a session after a login at the earliest. Not seen on a real Mac yet: no release until the owner asks.

## Technical Notes

- The fallback plist (`scheduler-open.plist`, `/usr/bin/open`) has the same registration and the same risk; the re-registration covers whichever is active.
- `heal()` runs at every launch; an unregister is async (`SMAppService.unregister`), so the re-registration waits for it, as `change` does in Settings.
- Debug and Release share one defaults domain and one `CURRENT_PROJECT_VERSION` (`UpdateModel.swift`), hence the path in the record; the path is also in `settings.json`'s `scheduler.app`.
- Two build records exist on purpose: this story's `SchedulerRegistration` (written only while scheduled scans are on, for the agent) and the story "Scan after an update when the saved report cannot be read"'s last launched build (written at every launch). Do not merge them.
- The owner's Mac is the only evidence so far; no test can reach launchd. The DDCKit tests cover the decision, the owner check covers launchd.

- **Research (2026-11-06; the critic checked each link).** No Apple documentation. The symptom matches in every report (an `SMAppService` agent that will not spawn after an app update while its status says enabled; `spawn failed`, exit 78, "needs LWCR update"). The repairs differ: [marinaMoji issue #41](https://github.com/marinaMoji/marinaMoji/issues/41) (checked once, same session), [Sanduhr PR #414](https://github.com/estevanhernandez-stack-ed/Sanduhr_f-r_Claude/pull/414) and [swishsaint PR #50](https://github.com/hectorpasserat2004-create/swishsaint/pull/50) (live checks pending) unregister and register; [multilink PR #9](https://github.com/pallewela/multilink/pull/9) did too and was reverted by their #13 ("Unregister/register never updates that pin"), moving to a per-version label (an ad-hoc signed daemon, so maybe not like ours); [diskwatch PR #4](https://github.com/valmayaki/diskwatch/pull/4) found Background Task Management re-submits a once-registered agent at login even after `unregister()`, and also moved to a new label; [arugula PR #877](https://github.com/arugula-salad/arugula/pull/877) is about a moved app, and notes `launchctl kickstart -k` hangs on a failing spawn. None checked after a logout or a restart. An Apple forum thread on a wrong parent bundle id ([727785](https://developer.apple.com/forums/thread/727785)) is a different cause. What broke the owner's registration is unknown (builds 10 to 17 ran under build 9's).
- **Approval after a re-registration** (the critic, 2026-11-06): if it ends in `.requiresApproval`, scans stop silently today (the footer still says "Next scan …"). The second pull request's "not running" line covers `.requiresApproval` too, with Settings → Permissions' Login Items route, and its AC says so.
- **A Homebrew user with the app quit** is not covered by the first half: `brew upgrade` does not launch the app, so the registration is repaired only when the app is opened (menu bar ADR §5, amended).
- If the owner check shows a Login Items notice or a new approval at every update, re-registering moves from "every new build" to "only when the not-running line shows" (Fix), and the ADR is amended again. If the repair does not survive a logout or a restart, the bundled plist moves to a per-build label (plan C, as multilink and diskwatch did).

## Dependencies

- Menu bar ADR "Menu bar app: Swift shell over the ddc CLI" §5 (the scheduler agent and its healing at every launch).
- Before "Scan after an update when the saved report cannot be read", whose scan runs on the tick.

## Attachments

-

---

*Last updated: 2026-11-06*

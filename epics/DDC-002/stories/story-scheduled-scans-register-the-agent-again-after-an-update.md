---
type: story
id: "story-scheduled-scans-register-the-agent-again-after-an-update"
epic: "DDC-002"
title: "Scheduled scans: register the agent again after an update"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-04
updated: 2026-11-04
external_refs: {}
tags: [app, scheduler, updates, bug]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Scheduled scans: register the agent again after an update

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner, 2026-11-04: the popover says "Next scan within the hour", but no scheduled scan has run on their Mac since 2026-10-31. The app log's last `tick:` line is from 0.11.1; every launch since, from 0.12.1 to 0.34.0, logs "scheduler: com.smartandpoint.deepdrivecleaner.scheduler.plist enabled". `launchctl print` of the agent says `job state = spawn failed`, `last exit code = 78: EX_CONFIG`, `properties = … resolve program`, and `parent bundle version = 9`, while the installed app is build 96. So launchd keeps the registration of an old build and cannot start the bundle program of the new one, and `SMAppService.status` still answers `.enabled`, so `SchedulerAgent.heal()`, whose `register` is idempotent, changes nothing. Each hourly tick fails before the app runs, and nothing tells the person.

What it breaks: no scheduled scan and no notification; no `ddc space` check on the tick; the saved report stays as old as the last scan by hand (0.29.1 on that Mac), so after each report schema change the popover says "No scan yet" (story "Scan after an update when the saved report cannot be read"), and "Grew since …" (story "ddc changes: what grew between the last two scans", DDC-001) has no pair to compare.

The cause is known only as far as `launchctl` shows it: when and why the registration stopped following the bundle is not known (a Sparkle update replaces the bundle in place; build 9 is far older than 0.11.1, which still ticked). The plan gate starts with Apple's documentation and known reports on `SMAppService` agents after an app update.

What the app should do:

- **Register again after an update in place.** `heal()` records the bundle path and the build it registered (`AppPreferences`, `SchedulerRegistration` = {path, build}; written only while scheduled scans are on). At a launch from the **same path** with **another build** (an update in place, by Sparkle or Homebrew), it unregisters the agent and registers it again, then records both. A launch from **another path** (a Debug build in DerivedData, a copy elsewhere) does not re-register: it logs it and keeps today's idempotent `register`, so a Debug run never moves the agent onto a bundle that may be deleted. Same path, same build: as today. **With no record yet** (the first launch of the build that ships this, the owner's case), the path comes from `settings.json`'s `scheduler.app`, which `heal()` writes while the agent is `.enabled`: the same path re-registers and records; another path only logs and registers as today. When the re-registration ends in `.requiresApproval` or the unregister throws, the record is still written (no loop at every launch) and Settings shows the state as today.
- **Say when the agent's ticks do not arrive.** The tick records when an **agent's** tick ran (`LastAgentTickAt`): a `--scheduled` launch, or a reopen while the app runs (the agent's `open` arrives as a reopen; a click on the Dock icon of a menu bar app is rare noise). The launch tick of the story "Scan after an update when the saved report cannot be read" does not record it. With scheduled scans on and the agent `.enabled`, when two hourly ticks are missing, counted in awake time: an agent tick records `ProcessInfo.systemUptime` beside `LastAgentTickAt` (on macOS it does not advance in sleep); "not running" is `systemUptime − tickUptime > 2 h`; when the saved uptime is larger than the current one, the Mac restarted since, and the rule is "no agent tick since boot and `systemUptime > 2 h`". So neither a reboot nor a long sleep counts, the popover's next-scan line and Settings → Schedule say "Scheduled scans are not running" with a **Fix** action that runs the re-registration. If ticks still do not arrive after Fix, the line offers the existing `SchedulerAgent.refused` route (Login Items, or `ddc schedule install`).
- When a `--scheduled` instance itself unregisters its job, launchd may end that instance; the cost is a lost log line, since the next launch heals again.

## Decomposition

- [ ] Research: `SMAppService` agents after an app update: what launchd keeps (a bundle path, a version, the signature), why `status` says `.enabled` while the job cannot spawn, and whether `unregister()` + `register()` keeps the approval and shows no "Background Items Added" notice. Start points: build 9 is 0.4.1 (`git log -S "CURRENT_PROJECT_VERSION = 9;"`); 0.11.1 (build 17) still ticked and 0.12.1 (build 19) did not, and between the two only the version changed in `app/`, `Makefile`, `scripts/` and `packaging/`, so the trigger is outside the code (how the update was installed, by Homebrew or Sparkle, per `app.log`; the macOS version; the signature)
- [ ] `SchedulerAgent.heal()`: the {path, build} record and the re-registration after an update in place
- [ ] `ScheduledWork`: `LastAgentTickAt` from agent ticks only; `NextScan`: a "not running" state from it and the uptime
- [ ] `PopoverView` and Settings → Schedule: the line, Fix, and the refused route after Fix
- [ ] Menu bar ADR §5 and the Sparkle ADR's line "heal() re-registers the scheduler agent at launch": a dated amendment each (idempotent `register` does not cover an update in place)
- [ ] DDCKit tests with the fake `AgentService`

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test (DDCKit, fake `AgentService`): a launch from the recorded path with another build, scheduled scans on, unregisters then registers the primary plist and records {path, build}; the same build only registers; another path does not re-register and logs it; with no record, a bundle path equal to `settings.json`'s `scheduler.app` re-registers and records, and another path does not; scheduled scans off unregister as today and record nothing; a re-registration ending in `.requiresApproval` or an unregister that throws still records, and the next launch of that build does not re-register
- [ ] Test (DDCKit): `NextScan` is "not running" when no agent tick ran for more than two hours of uptime with scheduled scans on and the agent enabled; an agent tick resets it, the launch tick does not; right after a launch with less than two hours of uptime it is not "not running"; a long sleep after the last tick does not count (uptime against uptime); after a restart, only awake time since boot counts; Fix runs the re-registration, and after a Fix with still no tick the line offers the refused route
- [ ] `make ddckit-test` passes and the Gate's `app` job builds the app
- [ ] The two ADR amendments are written and the critic has read them
- [ ] Owner check (runbook "Owner checks: manual acceptance across stories"): after the update that ships this, `launchctl print` of the agent shows the new build and no "spawn failed", the app log has a `tick:` line within the hour, macOS showed no Login Items notice, and the agent is still allowed; `launchctl print` of the login item (`SMAppService.mainApp`) is checked the same way

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- The fallback plist (`scheduler-open.plist`, `/usr/bin/open`) has the same registration and the same risk; the re-registration covers whichever is active.
- `heal()` runs at every launch; an unregister is async (`SMAppService.unregister`), so the re-registration waits for it, as `change` does in Settings.
- Debug and Release share one defaults domain and one `CURRENT_PROJECT_VERSION` (`UpdateModel.swift`), hence the path in the record; the path is also in `settings.json`'s `scheduler.app`.
- Two build records exist on purpose: this story's `SchedulerRegistration` (written only while scheduled scans are on, for the agent) and the story "Scan after an update when the saved report cannot be read"'s last launched build (written at every launch). Do not merge them.
- The owner's Mac is the only evidence so far; no test can reach launchd. The DDCKit tests cover the decision, the owner check covers launchd.

## Dependencies

- Menu bar ADR "Menu bar app: Swift shell over the ddc CLI" §5 (the scheduler agent and its healing at every launch).
- Before "Scan after an update when the saved report cannot be read", whose scan runs on the tick.

## Attachments

-

---

*Last updated: 2026-11-04*

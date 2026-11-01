---
type: story
id: "story-windows-scheduled-scans-through-task-scheduler"
epic: "DDC-008"
title: "Windows scheduled scans through Task Scheduler"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-31
updated: 2026-11-01
external_refs: {}
tags: [windows, schedule, cli, platform]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Windows scheduled scans through Task Scheduler

| Field | Value |
|---|---|
| **Epic** | [DDC-008](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

On macOS, `ddc schedule install --interval daily|weekly [--threshold SIZE]` writes a LaunchAgent (`crates/ddc-cli/src/platform/launchd.rs`) that runs the background scan. `ddc schedule uninstall` and `ddc schedule status` remove and report it. On Windows the same commands manage a Task Scheduler task. The tray app's Schedule menu calls them, so there is one implementation, tested from the CLI.

- **The task.** One per-user task, `\DeepDriveCleaner\Scheduled scan`:
  - triggered daily or weekly at a fixed time;
  - "Run only when user is logged on", so that a notification can be shown;
  - "Run task as soon as possible after a scheduled start is missed", the counterpart of launchd's catch-up after sleep;
  - no administrator rights, and no stored password.
- **What the task runs.**
  - When the Windows app is installed: `DeepDriveCleaner.exe --scheduled` (amended 2026-11-01, [[adr/windows-app-tauri-2-shell-over-the-ddc-cli|Windows app: Tauri 2 shell over the ddc CLI]] §5). It runs `scan --background --notify json`, shows the notification and records it, as the macOS app does with its bundled agent.
  - Without the tray (a script or cargo install): `deepdrivecleaner.exe scan --background`. It saves `last-scan.json` and logs.
  
  `schedule install` chooses between them: it writes `DeepDriveCleaner.exe --scheduled` as the task's action when the CLI runs from the setup's `bin\` folder and `..\DeepDriveCleaner.exe` exists, and otherwise `<this binary> scan --background`. `schedule status` names the program and says whether it notifies.
  
  The bare CLI shows no notification on Windows (seam ADR §3, `notifications` absent; app ADR §4): its scheduled scan notifies nobody, and `schedule status` says so.
- **Hour, pause, remind and status for the app** (added 2026-11-01, app ADR §5): `schedule install --at <hour>` picks the hour (whole hours, as on macOS); `schedule pause` and `schedule resume` disable and enable the task, keeping its trigger; `schedule remind --at <time>` adds a one-shot trigger (the reminder scan after Remind me tomorrow with a weekly schedule); `schedule status --json` prints whether the task exists, is paused, the interval and hour, its next run time and last run time and result from the Task Scheduler API (`IRegisteredTask`, never localized `schtasks` text), and its action.
- **Capability.** Windows declares `schedule` once this lands. Until then `ddc schedule` refuses with exit 69, as the platform seam ADR requires.

## Decomposition

- [ ] A scheduler seam in `cmd/schedule.rs`: one trait, with launchd and Task Scheduler behind it, chosen by the platform module. `cmd/schedule.rs` imports launchd directly today. The help text of `schedule` becomes platform-neutral (it says "LaunchAgent" now).
- [ ] Decide how the task is written, and record it in an amendment of the app ADR: the Task Scheduler COM API through the `windows` crate, or `schtasks.exe /create /xml` with a generated task XML. The XML is easy to unit-test on every OS; COM avoids a child process. `status` reads the next run time through COM either way: the task XML holds the trigger, not the next run.
- [ ] `platform/windows/task_scheduler.rs` in `ddc-cli`, beside `launchd.rs`:
  - generate the task (unit-tested on every OS);
  - install, update, uninstall, status;
  - "installed and current" means the task's action is the one the rule above picks for this binary, with the requested interval.
- [ ] `schedule status` on Windows: whether the task exists, its next run time, its last run result, and whether anything will show a notification.
- [ ] Declare `schedule` in the Windows capability list.
- [ ] CI in the Gate's Windows job: install, query, update, uninstall. A forced run (`schtasks /run`) writes `last-scan.json`. If a hosted runner's session cannot run a "logged on only" task, the test runs the task's action line directly and says so.
- [ ] Add the skips this removes from `.config/native-skips/windows.txt`, and run the native Windows leg on the owner's lab machine.

## Implementation Plan

## Acceptance Criteria

- [ ] `ddc schedule install --interval weekly` on Windows creates the task. `schtasks /query /tn "\DeepDriveCleaner\Scheduled scan" /xml` shows a weekly trigger, `StartWhenAvailable` set, and the action the rule above picks. Installing again with `--interval daily` replaces it rather than adding a second task. The task's action is `..\DeepDriveCleaner.exe --scheduled` when this binary runs from the setup's `bin\` folder and that file exists, and otherwise this binary with `scan --background`. CI covers both by placing a stub `DeepDriveCleaner.exe` one folder above a copy of the CLI in `bin\` for one case.
- [ ] `ddc schedule uninstall` removes the task. `ddc schedule status` reports it absent (CI).
- [ ] `schedule pause` disables the task and `schedule resume` enables it with its trigger unchanged; `schedule remind --at` adds one one-shot trigger and a second call replaces it; `schedule status --json` reports each state and validates against its schema (CI).
- [ ] A forced run of the task writes a fresh `last-scan.json` under the Windows data folder (CI).
- [ ] The task XML generation has unit tests that run on macOS, Linux and Windows.
- [ ] On macOS, `schedule` behaves exactly as before: launchd tests and goldens unchanged.

## Final Summary

## Technical Notes

- The scan itself, and what it skips on Windows, belong to DDC-007's story "Windows scanner and rule catalog". This story only schedules it.
- A per-user task created by the user needs no elevation. A task set to run whether or not the user is logged on needs a stored password, and is not used.
- The time of day: the macOS LaunchAgent uses 11:30. `StartWhenAvailable` fires a missed run at sign-in, the busiest moment, so the trigger gets a delay of a few minutes.
- No unattended cleaning: the task only scans (the owner, 2026-10-28; story "Automatic cleaning of download caches after a scheduled scan", parked).

## Dependencies

- DDC-007 story "Windows scanner and rule catalog" (the background scan must run on Windows).
- The tray story's ADR, for the notification path.

## Attachments

-

---

*Last updated: 2026-11-01*

---
type: story
id: "story-windows-app-scheduled-scans-and-notifications"
epic: "DDC-008"
title: "Windows app: scheduled scans and notifications"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [windows, tauri, app, schedule, notify]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Windows app: scheduled scans and notifications

| Field | Value |
|---|---|
| **Epic** | [DDC-008](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

Scheduled scans and their notification in the Windows app ([[adr/windows-app-tauri-2-shell-over-the-ddc-cli|Windows app: Tauri 2 shell over the ddc CLI]] §5, §6). The schedule itself is the CLI's Task Scheduler task (story "Windows scheduled scans through Task Scheduler"); this story is the app's side.

- **`--scheduled`** never registers the single-instance plugin, so it never swallows a user launch. With a resident instance running (its named mutex), it starts `DeepDriveCleaner.exe --run-scheduled`, which the plugin forwards to the resident; the resident runs the scan through its queue and posts the toast. Without one, it runs headless (no tray, window or web view): `scan --background --notify json`, the working-hours rule, the toast, `notify ack`, exit.
- **The toast:** "12.4 GB can be freed", the top categories, **Review** and **Remind me tomorrow**. The rule, thresholds, snooze and low-disk rules are the CLI's.
- **Buttons by protocol activation:** `deepdrivecleaner:review` and `deepdrivecleaner:snooze`, so a click from the Action Center works after the posting process exited. `review` opens the Review window in a resident instance (forwarded, or a new one); `snooze` runs `notify snooze` and, with a weekly schedule, `schedule remind --at` for the next day's hour. The handler accepts only these two URIs.
- **Working hours** are the app's rule, ported from `DDCKit` `ScheduledWork`: a decision outside them is kept pending in the app's preferences folder and posted by a resident instance at the first moment inside them (one timer), or at the next start or scheduled run.
- **Pause and Resume** in the context menu run `schedule pause` / `schedule resume`; on Resume the app runs one scheduled scan at once if the last scheduled run is older than one interval. The panel shows the next run from `schedule status --json`.

## Decomposition

- [ ] `ddc-app-model`: `ScheduledWork` (working hours, pending decisions, the Resume catch-up rule), the notification text, the protocol URI parser (two verbs, anything else refused); ported tests.
- [ ] The `--scheduled` and `--run-scheduled` paths; the resident's named mutex; a `--run-scheduled` that finds no resident runs headless and never creates the tray.
- [ ] The toast from XML the shell builds (`XmlDocument::LoadXml`, `CreateToastNotifier(aumid).Show`; ADR §6), with `launch` and both buttons protocol-activated; the protocol registration for development builds (the setup registers it for installs).
- [ ] Pause and Resume, the next-run line.

## Implementation Plan

## Acceptance Criteria

- [ ] A model test covers each notify decision and the text it produces.
- [ ] CI on Windows: `--scheduled` against the fake CLI exits 0 and posts nothing when the decision is "do not notify"; with a resident instance running, it starts no scan itself and the resident's queue runs exactly one.
- [ ] A test of the protocol handler: `deepdrivecleaner:review` and `deepdrivecleaner:snooze` are accepted, any other argv is refused whole (accepted only as exactly `[exe, URI]`, and `--run-scheduled` only as `[exe, "--run-scheduled"]`), and no URI leads to a removing command (the argv allowlist).
- [ ] Manual (owner, lab machine): a forced task run shows the toast; Remind me tomorrow, clicked from the Action Center after the run exited, suppresses the next one; Review opens the Review window with and without a resident instance; Pause disables the task and Resume enables it.

## Dependencies

- Story "Windows scheduled scans through Task Scheduler".
- Story "Windows app: Tauri shell, tray icon and the panel".
- The setup (story "Windows installer: one signed setup with the CLI and the tray") for the Start menu shortcut with the AUMID; a development build registers its AUMID under `HKCU\Software\Classes\AppUserModelId` instead (ADR §4).

## Final Summary

## Attachments

-

---

*Last updated: 2026-11-01*

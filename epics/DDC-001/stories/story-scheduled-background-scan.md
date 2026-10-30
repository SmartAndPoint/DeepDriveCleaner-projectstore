---
type: story
id: "story-scheduled-background-scan"
epic: "DDC-001"
title: "Scheduled background scan"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-17
updated: 2026-10-19
external_refs: {}
tags: [background, launchd]
code_refs: ["crates/ddc-cli/src/platform/launchd.rs", "crates/ddc-cli/src/notify.rs", "crates/ddc-cli/src/cmd/background.rs", "crates/ddc-cli/src/cmd/schedule.rs", "crates/ddc-cli/src/cmd/scan.rs", "crates/ddc-cli/src/cmd/mod.rs", "crates/ddc-cli/src/cli.rs", "crates/ddc-cli/src/main.rs", "crates/ddc-cli/tests/background_cli.rs", "crates/ddc-cli/tests/structure.rs", "crates/ddc-core/src/report/mod.rs", "crates/ddc-core/src/rules/visitor.rs"]
specs: []
started_at: "2026-10-17T10:44:36.821Z"
closed_at: "2026-10-17T11:52:25.555Z"
plan_updated_at: "2026-10-17T10:44:36.821Z"
---

# Scheduled background scan

| Field | Value |
|---|---|
| **Epic** | [DDC-001](../epic.md) |
| **Status** | done |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

A periodic scan without user involvement: save the report and notify if a lot of junk has piled up. Per the user's decision and [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]], background mode **deletes nothing** and, without Full Disk Access, does not enter TCC-protected regions.

## Decomposition

- [x] `ddc schedule install [--interval daily|weekly] [--threshold 10G]`, `ddc schedule uninstall`, `ddc schedule status`
- [x] LaunchAgent `~/Library/LaunchAgents/com.smartandpoint.deepdrivecleaner.plist` (hand-written XML, with escaping): `StartCalendarInterval`, `ProcessType=Background`, `LowPriorityIO`, `Nice`, `SoftResourceLimits.NumberOfFiles`, `StandardOutPath`/`StandardErrorPath`; `ProgramArguments[0]` is the canonical `current_exe()`
- [x] `install` refuses if `current_exe()` is inside a `target/` directory (`ddc clean` itself may delete it), and suggests `cargo install --path crates/ddc-cli` — implemented for a `target/` next to a `Cargo.toml` (`launchd::inside_cargo_target`); the hint names `make install`
- [x] `install`: write the plist atomically → `launchctl bootout gui/$UID/<label>` (ignore the "not loaded" error) → `launchctl bootstrap gui/$UID <plist>`; `uninstall`: `bootout` → delete the plist; `status`: `launchctl print gui/$UID/<label>` + compare `ProgramArguments[0]` with `current_exe()` + age of the last scan
- [x] `ddc scan --background`: no terminal output, 2–4 threads, without Full Disk Access does not open regions where macOS shows a prompt (`~/Desktop`, `~/Documents`, `~/Downloads`, `~/Pictures`, `~/Movies`, `~/Music`, `~/Library/Containers`, `~/Library/Group Containers`, Mail, Messages, Calendars, Reminders, Contacts, call history, iPhone backups), with Full Disk Access scans everything; exits if the lock is held; report to `last-scan.json`; notification via `osascript` (`on run argv`, no string interpolation into the script) if the amount that can be freed ≥ threshold (10 GB) or free space < 10%; repeat only on growth of ≥ 20% or after ≥ 7 days
- [x] Without Full Disk Access, the background scan does not open regions where macOS asks the user (otherwise the read blocks until the prompt is answered), does not crash, and lists the inaccessible paths; with Full Disk Access it scans everything. Binary `deepdrivecleaner`, agent label `com.smartandpoint.deepdrivecleaner` — evidence: `crates/ddc-cli/tests/background_cli.rs` `tcc_areas_are_never_opened`, `with_full_disk_access_prompting_areas_are_scanned`
- [x] Background-path modules do not import `plan` or `exec`

## Implementation Plan

Route per the `projectstore:planner` plan (section "Background scan") and ADR "Cleanup safety model: risk classes, plan and confirmation" §8: the background mode only scans, skips TCC regions if there is no Full Disk Access, and exits if the lock is held.

1. **Core** — `ScanOptions.skip` (paths that are not walked: `Pre::Skip`, the directory is not read, so no TCC prompts); `sys::procs::uid()`.
2. **`crates/ddc-cli/src/launchd.rs`** — hand-written plist with XML escaping (`Label` `com.smartandpoint.deepdrivecleaner`, `ProgramArguments` = canonical `current_exe` + `scan --background --threshold N`, `StartCalendarInterval` for daily/weekly, `ProcessType=Background`, `LowPriorityIO`, `Nice`, `SoftResourceLimits.NumberOfFiles`, `StandardOutPath`/`StandardErrorPath` into the log directory); a `Launchctl` trait (`bootout`/`bootstrap`/`print`) with a real implementation via `/bin/launchctl` and a fake in tests; atomic plist write (`store::write_atomic`).
3. **`cmd/schedule.rs`** — `install [--interval daily|weekly] [--threshold 10G]`: refuse if the binary is in `target/` (hint: `cargo install --path crates/ddc-cli`); write the plist → `bootout` (the "not loaded" error is ignored) → `bootstrap`; `uninstall`: `bootout` → delete the plist; `status`: plist present or not, agent loaded or not, whether `ProgramArguments[0]` matches the current binary, age of the last scan.
4. **`notify.rs`** — a pure decision function: notify if "can be freed" ≥ threshold or free space < 10%; notify again only on growth of ≥ 20% or after ≥ 7 days (state in `<data-dir>/notify-state.json`); send via `osascript` with `on run argv` (the text is not interpolated into the script).
5. **`cmd/background.rs`** (`ddc scan --background`, hidden `--no-notify`, `--threshold`) — no terminal output; if the lock is held, write a line to the log and exit 0; 3 threads; skip regions where macOS shows a prompt only without Full Disk Access (list as in ADR §8; access is detected by whether `~/Library/Application Support/com.apple.TCC/TCC.db` is readable); `last-scan.json`; notification; a line in `<log-dir>/background.log`. Background-path modules do not import `plan` or `exec`.
6. **Tests** — `plutil -lint` of the generated plist; `install`×2 / `uninstall`×2 with a fake `launchctl`; refusal from `target/`; notification logic; integration `ddc scan --background --no-notify` (tree unchanged, no `actions.jsonl`, `last-scan.json` present, TCC regions not read); held lock; structural test of the background-path sources.
7. **Manual check** — `launchctl kickstart -k gui/$UID/com.smartandpoint.deepdrivecleaner` after installing via `cargo install`.

Files touched: `crates/ddc-core/src/{report/mod.rs,rules/visitor.rs,sys/procs.rs}`, `crates/ddc-cli/src/{launchd.rs,notify.rs,cmd/background.rs,cmd/schedule.rs,cmd/scan.rs,cli.rs}`, `crates/ddc-cli/tests/{background_cli.rs,structure.rs}`.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [x] Test: `plutil -lint` passes on the generated plist — evidence: `crates/ddc-cli/src/launchd.rs` `generated_plist_passes_plutil`; live `plutil -lint ~/Library/LaunchAgents/com.smartandpoint.deepdrivecleaner.plist` OK (2026-10-17)
- [x] Test with a fake `launchctl`: `install` twice and `uninstall` twice in a row complete without errors, and there is one agent — evidence: `launchd.rs` `install_and_uninstall_are_idempotent`, `a_failed_bootout_keeps_the_agent_and_its_plist`
- [x] Test: `install` from a binary inside `target/` refuses with a hint — evidence: `launchd.rs` `a_cargo_build_is_never_scheduled`
- [x] Integration test: after `ddc scan --background --no-notify` over a temporary tree, paths, types and inodes are unchanged, `actions.jsonl` is not created, `last-scan.json` is written — evidence: `crates/ddc-cli/tests/background_cli.rs` `background_scan_only_reads_and_saves_the_report` (path → inode and file type snapshot)
- [x] Structure test: background-path sources (`cmd/background.rs`, `notify.rs`, `launchd.rs`) do not reference `exec::`, `plan::`, `Executor`, `trash`, `remove_dir_all` — evidence: `crates/ddc-cli/tests/structure.rs` `background_sources_never_reference_cleanup`
- [x] Test: `--background` with the lock held exits without scanning — evidence: `background_cli.rs` `background_scan_steps_aside_while_the_lock_is_held`
- [x] Test: repeat-notification logic (threshold, growth ≥ 20%, ≥ 7 days) and notification text (size + `ddc clean`) — evidence: `crates/ddc-cli/src/notify.rs` `threshold_or_low_free_space_triggers`, `repeats_only_after_growth_or_a_week`, `message_names_the_size_and_the_command`
- [x] Manual check: the notification is actually shown from the launchd context (`launchctl kickstart -k gui/$UID/<label>`) — evidence: 2026-10-17 `launchctl kickstart -k gui/501/com.smartandpoint.deepdrivecleaner` → `background.log` "scanned 5109091 files in 241.4s; can free ~88.1 GiB; inaccessible 46; notified true"; the user confirmed the notification on screen

## Final Summary

**What changed.** `ddc schedule install [--interval daily|weekly] [--threshold 10G]`, `uninstall` and `status` (`crates/ddc-cli/src/cmd/schedule.rs`). A LaunchAgent plist written by hand with XML escaping and saved atomically (`launchd.rs`): `StartCalendarInterval`, `ProcessType=Background`, `LowPriorityIO`, `Nice`, `SoftResourceLimits.NumberOfFiles`, log paths, and the canonical binary as `ProgramArguments[0]`; a build inside a cargo `target/` is refused. Install is bootout → bootstrap; bootout ignores only "not loaded", and uninstall keeps the plist while the job is still loaded. `ddc scan --background` (`cmd/background.rs`) is silent, uses 3 threads and steps aside while `ddc clean` holds the lock. Without Full Disk Access it does not open the folders where macOS would show a prompt (`ScanOptions.skip`, applied in `RuleVisitor` before a directory is read), because a prompt blocks the read. It saves `last-scan.json`, notifies through `osascript` with the text passed as arguments (`notify.rs`) and appends a line to `background.log`. The binary is `deepdrivecleaner`; the agent label is `com.smartandpoint.deepdrivecleaner`.

**Why.** ADR "Cleanup safety model: risk classes, plan and confirmation" §8: the background mode only scans and notifies; the user cleans interactively. The prompt skip came from a live hang: a background scan without Full Disk Access blocked in `open()` on `~/Library/Containers` until the prompt was answered.

**Tests.** `make check`: 108 tests, fmt and clippy clean (2026-10-17). For this story: `crates/ddc-cli/src/launchd.rs` (4), `notify.rs` (3), `crates/ddc-cli/tests/background_cli.rs` (4), `crates/ddc-cli/tests/structure.rs` (1). `projectstore:reviewer` found no blockers; its should-fix items were fixed and tested: bootout now fails on anything but "not loaded" (`a_failed_bootout_keeps_the_agent_and_its_plist`), and a zero baseline no longer defeats the repeat rule (`repeats_only_after_growth_or_a_week`). Also from the review: a notification state that cannot be saved is logged instead of failing the run, the Full Disk Access probe is shared with the report (`report::full_disk_access`), the refusal hint names `make install`, and the inode snapshot also compares file types.

**Live run (2026-10-17).** `ddc schedule install --threshold 10G`, then `launchctl kickstart -k gui/$UID/com.smartandpoint.deepdrivecleaner`: `background.log` recorded "scanned 5109091 files in 241.4s; can free ~88.1 GiB; inaccessible 46; notified true", and the user confirmed that the notification appeared on screen. `plutil -lint` on the installed plist passes; `launchctl print` shows the job loaded with last exit code 0.

**Risks and follow-ups.**
- The binary is ad-hoc signed, so its Full Disk Access grant is tied to one build and is lost on every `make install`. Developer ID signing (or a stable signing identity) fixes it. The installed binary predates this story's review fixes and the walker's `EINTR` retry; it was not rebuilt, to keep the grant.
- Behaviour with Full Disk Access is covered by `with_full_disk_access_prompting_areas_are_scanned` (fake `TCC.db`); a launchd `du` run with the real grant read `AddressBook` and `Group Containers` without prompts, but no scheduled scan with the grant has run yet.
- `status` prints the last scan's timestamp, which may come from a foreground scan; printing its age and the last `background.log` line is a follow-up.
- No watchdog: a background scan that blocks keeps the shared lock, and `ddc clean` then reports that a scan or clean is running.
- `DDC_DATA_DIR` set at install time is not passed to the job; the prompt skip matches exact paths, so `ddc scan --background <path inside Documents>` would open it (the agent never passes paths); prompt-area skips are reported with the reason `excluded`.

**Superseded for low disk (2026-10-19).** In epic DDC-002 low disk is no longer held back by the repeat rule (ADR "Menu bar app: Swift shell over the ddc CLI" §6 item 9). The tests cited above are now `the_threshold_repeats_only_after_growth_or_a_week` and `message_names_the_size_the_command_and_a_full_volume`.

## Technical Notes

- `osascript display notification` from a LaunchAgent is shown as coming from Script Editor; on a fresh system you may need to allow notifications for Script Editor. In the Mac app we will replace it with UserNotifications.
- A LaunchAgent does not inherit the terminal's Full Disk Access; the soft limit on open files under launchd is 256.

## Dependencies

- [[epics/DDC-001/stories/story-ddc-scan-report|ddc scan report]]

## Attachments

-

---

*Last updated: 2026-10-19*

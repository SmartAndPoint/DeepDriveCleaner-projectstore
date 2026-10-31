---
type: story
id: "story-windows-tray-app-a-minimal-scheduling-icon"
epic: "DDC-008"
title: "Windows tray app: a minimal scheduling icon"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-31
updated: 2026-10-31
external_refs: {}
tags: [windows, tray, app, schedule, notify, adr]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Windows tray app: a minimal scheduling icon

| Field | Value |
|---|---|
| **Epic** | [DDC-008](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner wants "a minimal daemon icon for scheduling in the Windows system tray" (2026-10-31). It is the Windows counterpart of the macOS menu bar app, at its smallest. It shows the last result, starts a scan and sets the schedule, and it notifies after a scheduled scan. Reviewing and cleaning stay in the CLI for now. A Windows review window is a later story, if dogfooding asks for one.

**What the user sees**

- An icon in the notification area. It follows the taskbar's light or dark theme, and has three states:
  - idle;
  - scanning;
  - something to free, shown as a small accent dot.
- The tooltip: "DeepDriveCleaner · 12.4 GB can be freed · scanned today 10:02", or "Scanning…", or "Not scanned yet".
- A left or right click opens one menu:
  - a status line, not clickable: the tooltip's text;
  - **Scan now** (disabled while a scan runs);
  - **Scan in a terminal…** opens a terminal running `ddc scan`, whose output ends with the `ddc clean` hint. It runs a new scan: no CLI command prints the saved report yet;
  - **Schedule ▸** Daily · Weekly · Off, a radio group. It calls `ddc schedule install --interval …` or `ddc schedule uninstall`. It shows the next run time when one is set;
  - **Start at sign-in**, a check mark;
  - **About DeepDriveCleaner** (version, website, licence);
  - **Quit**.
- After a scheduled scan whose decision says to notify, a Windows notification:
  - "12.4 GB can be freed", with the top categories in the body;
  - **Show** opens the same terminal (a new scan);
  - **Remind me tomorrow** calls `notify snooze`.
  
  It has the same decision rule, thresholds, snooze and low-disk rules as macOS. The rule stays in Rust ([[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]], notifications and notify state).

**What it never does.** It never deletes, cleans or empties anything, and it has no "clean automatically" option (the owner, 2026-10-28: scans inform, people remove).

**How it works**

- `ddc-tray.exe` runs as a GUI-subsystem process, so no console window appears.
- One resident instance per user, held by a named mutex. A second resident start brings up the menu of the running one and exits. The mutex covers the resident mode only.
- Two modes:
  - **resident** (started at sign-in or from the Start menu): the icon and the menu, kept current from `last-scan.json` and the scan's progress events;
  - **`--scheduled`** (started by the Task Scheduler task): it runs `scan --background --notify json`, shows the notification if the decision says so, and records it with `notify ack`. Then it tells a running resident instance to refresh. If a resident instance is already scanning (Scan now), the scheduled run skips its own scan, so two scans never write `last-scan.json` at once.
  - A notification's buttons reach a process only while it is alive, or through a registered COM activator. So `--scheduled` stays alive until the notification is dismissed, activated or expires, as the macOS `--scheduled` instance lives about a minute. It may instead hand the notification to a running resident instance. COM activation is used only if the ADR finds it cheap.
- It is a shell over the CLI. It runs the `deepdrivecleaner.exe` installed next to it, with `--json`, and decodes the same schemas as the macOS app. No core logic is duplicated.
- Budget: idle CPU near zero, a few MB of memory, no polling. It wakes on a file change notification or a timer for the schedule line.

## Decomposition

- [ ] ADR "Windows tray app: Rust shell over the ddc CLI", modelled on the macOS app's ADR. Run the critic on it. It decides:
  - the language and crates. The proposal is Rust in the workspace as `crates/ddc-tray` (Windows-only binary, platform-neutral model module), using the `tray-icon` and `muda` crates or `Shell_NotifyIconW` through the `windows` crate. Notifications go through WinRT `ToastNotificationManager`;
  - the AppUserModelID and how it is registered: a Start menu shortcut written by the installer, or a key under `HKCU\Software\Classes\AppUserModelId`. Also whether the bare CLI may use the same ID;
  - the IPC between the `--scheduled` run and the resident instance (a window message or a named event);
  - how a notification's buttons reach the app when it is not running (activation by the AUMID).
- [ ] The icon: a 16/20/24/32 px glyph in light and dark variants plus the dot. It is derived from the app icon (`scripts/icons/`), so the two products look alike.
- [ ] A platform-neutral `TrayModel`: last scan, schedule and progress turned into the tooltip, the menu items and their enabled states, and the notification text. Unit-tested on every OS.
- [ ] The Windows shell: icon, menu, theme switch, single instance, Start at sign-in (`HKCU\...\Run`), the `--scheduled` mode, notifications and their actions.
- [ ] `crates/ddc-tray` joins the gate: the coverage floor, `deny.toml` licences for its crates (`tray-icon`, `muda`, `windows`), and the complexity guard. It builds on every target of `cargo check --workspace`, with a stub `main` off Windows.
- [ ] CI in the Gate's Windows job: the binary starts, a second start exits 0 and leaves one instance, `--scheduled` against a fake CLI writes nothing and exits 0 when the decision is "do not notify".
- [ ] Dogfooding on the owner's lab machine, with the owner's go-ahead for the scans it runs.

## Implementation Plan

## Acceptance Criteria

- [ ] The ADR exists and is critic-approved.
- [ ] `TrayModel` tests cover each state (not scanned, scanning, nothing to free, something to free, scan failed, schedule off/daily/weekly) and each notification decision. They run in the macOS gate.
- [ ] The CI checks in the decomposition pass on `windows-latest`.
- [ ] No code path in `ddc-tray` runs `clean`, `disk free` or any other command that removes files. A test lists the CLI subcommands the tray may run and fails on any other.
- [ ] Manual (owner, lab machine), recorded in the owner-checks runbook:
  - the icon is visible and legible on light and dark taskbars;
  - Scan now updates the tooltip;
  - Schedule ▸ Weekly creates the task, and Off removes it. The next run time comes from the task's XML, not from localized `schtasks` text;
  - a forced scheduled run shows the notification;
  - Remind me tomorrow suppresses the next one, also when no resident instance is running;
  - Quit and Start at sign-in behave as labelled.

## Final Summary

## Technical Notes

- The macOS menu bar app is Swift, because native look matters there and SwiftUI gives it cheaply. On Windows the tray needs only an icon, a menu and a notification. These are Win32 and WinRT calls that Rust reaches directly. A Rust tray keeps one toolchain and one gate, and needs no .NET runtime. The ADR confirms or rejects this.
- A Store MSIX (if the distribution ADR picks it) would declare the sign-in start as a `StartupTask` and get its AUMID from the package. A setup.exe registers both itself. The ADR covers both cases.

## Dependencies

- DDC-007 story "Windows scanner and rule catalog".
- Story "Windows scheduled scans through Task Scheduler".
- Story "Windows signing and distribution route: research and ADR", for the AUMID and sign-in registration under the chosen packaging.

## Attachments

-

---

*Last updated: 2026-10-31*

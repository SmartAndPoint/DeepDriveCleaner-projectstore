---
type: story
id: "story-windows-app-tauri-shell-tray-icon-and-the-panel"
epic: "DDC-008"
title: "Windows app: Tauri shell, tray icon and the panel"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [windows, tauri, app, tray, rust]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Windows app: Tauri shell, tray icon and the panel

| Field | Value |
|---|---|
| **Epic** | [DDC-008](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The first slice of the Windows app ([[adr/windows-app-tauri-2-shell-over-the-ddc-cli|Windows app: Tauri 2 shell over the ddc CLI]]). The owner widened the minimal tray icon this story first planned into a full counterpart of the macOS app, built with Tauri 2 (2026-11-01). This story lays the ground the other app stories build on, and ships the part a person sees first: the tray icon and the panel.

It replaces the earlier plan "Windows tray app: a minimal scheduling icon" (this file, renamed). Its rules carry over: the app never deletes on its own, it is a shell over the CLI, one resident instance per user, no polling.

**What the user sees**

- A tray icon in the notification area, in light and dark variants for the taskbar theme, with the macOS states: idle, scanning, paused (grey), and a dot for "needs you" or after a notification.
- The tooltip: "DeepDriveCleaner · 12.4 GB can be freed · scanned today 10:02", or "Scanning…", or "Not scanned yet".
- A left click opens **the panel** above the icon, the counterpart of the macOS popover (`PopoverView`):
  - the status line and the last scan's time;
  - the free space of the home's volume as a bar;
  - what the last scan found, grouped as the macOS popover groups it, with sizes;
  - **Scan now**, with progress from the CLI's events and **Stop**;
  - **Review…** and **Clean Selected…** (open the Review window and the confirmation of story "Windows app: Review window and clean with confirmation"; hidden until that story lands);
  - the "saved scan is not of the home folder" note, as on macOS.
- A right click opens the native context menu: Scan now, Pause scans or Resume scans, Settings…, About DeepDriveCleaner, Quit. Pause and Settings are disabled until their stories land.
- Escape or a click elsewhere hides the panel.

**How it works**

- `crates/ddc-app-model` in the root workspace (serde only, platform-neutral), and `app-windows/`, a separate Cargo workspace for the Tauri shell (`deepdrivecleaner-app`) and `ui/` (static HTML, CSS and ES modules, no Node) (ADR §2).
- `ddc-app-model` gets what this slice needs, ported from `DDCKit`: the scan report types (`ScanReport.swift`), formatting (`Formatting.swift`), the panel model (`PopoverModel.swift`) and the runner's queue and lanes (`CLIRunner.swift`, macOS ADR §1 and amendments). Each ported rule carries its `DDCKit` test over as a Rust test.
- The shell spawns `bin\deepdrivecleaner.exe` beside the app with `--json` and `CREATE_NO_WINDOW`, reads `last-scan.json` at start and after each run, and watches the data folder for changes made from a terminal.
- `tauri-plugin-single-instance`: a second start shows the panel of the running instance and exits. The resident instance also holds the named mutex `--scheduled` checks (ADR §5).
- A click on the tray icon within 300 ms of the panel hiding on focus loss does not reopen it.
- The app's own icon and the tray glyphs come from `scripts/icons/`, so the two apps look alike; the `.ico` files are committed, because that generator is Swift.

## Decomposition

- [ ] ADR "Windows app: Tauri 2 shell over the ddc CLI", critic-approved.
- [ ] `cargo deny` on a Tauri 2 hello world; the licence list for `app-windows/deny.toml`, with the owner's answer on MPL-2.0, recorded in the ADR before the shell merges.
- [ ] Gate: `crates/ddc-app-model` joins the root workspace (it then runs in every Rust push); `app-windows/` gets `make win-app-test` and `make win-app-build`, a fourth `gate-scope.sh classify` answer `win-app` with its pre-push branch and `test_gate_scope.py` cases, and a Gate job `win-app` on `windows-2025` that runs on every push. `tauri-cli` joins `make tools`. The owner adds the job to `main`'s required checks.
- [ ] **A parity inventory** in this story: every file of `app/Packages/DDCKit/Sources/DDCKit/` mapped to the Windows story that ports it, or "no Windows meaning" with the reason, or "later". The other app stories' parity tables refer to it.
- [ ] `ddc-app-model`: report types decoded from the CLI's goldens and the `DDCKit` fixtures, formatting, the panel model, the runner rules, a timer registry.
- [ ] The shell: tray icon and states, the panel window placed by the positioner (`TrayCenter`, constrained to the work area), the context menu, single instance, the runner spawning the CLI, Stop through `--cancel-event` (ADR §1), the data-folder watch, the AUMID registry entry for development builds.
- [ ] `ui/`: the panel in Fluent-like CSS, light and dark, keyboard and Narrator labels.
- [ ] A fake CLI for the shell's tests (a test binary that prints recorded reports and events), so the app is built and tested before the Windows scanner is supported.
- [ ] A test lists the CLI subcommands and flags the app may run and fails on any other (ADR §8).

## Implementation Plan

## Acceptance Criteria

- [ ] The ADR exists and is critic-approved.
- [ ] `make win-app-test` passes on macOS, Linux and Windows; it covers each panel state (not scanned, scanning, stopped, nothing found, found, scan failed, report of another folder, report from an older schema).
- [ ] The Gate's `win-app` job builds `DeepDriveCleaner.exe` on Windows; the shell's command handlers, tested with Tauri's mock runtime (`tauri::test`) against the fake CLI, run `scan --json` once when Scan now is invoked twice; a second process start leaves one instance.
- [ ] The parity inventory covers every `DDCKit` source file.
- [ ] No code path runs `clean`, `disk free` or another removing command (the allowlist test).
- [ ] With the panel hidden, the timer registry holds no timer under 60 seconds and the shell polls no file (a test of the registry, ADR §9).
- [ ] Manual (owner, recorded in the owner-checks runbook): the icon is legible on light and dark taskbars; the panel opens above the icon with the taskbar at the bottom and at the top; Scan now updates the panel and the tooltip; Narrator reads the panel's controls.

## Technical Notes

- Develop on macOS with `cargo tauri dev` in `app-windows/` (Tauri runs there on WKWebView); build and test the Windows specifics on the owner's Windows machine over ssh, as the CLI's native runs do. `native-windows.sh` gains an option for the app workspace.
- WebView2 is present on Windows 11; the shell checks for it at start and says so if it is missing.

## Dependencies

- DDC-007 story "Windows scanner and rule catalog" for real scans; the slice is built against the fake CLI before it.

## Final Summary

## Attachments

-

---

*Last updated: 2026-11-01*

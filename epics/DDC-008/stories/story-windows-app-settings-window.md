---
type: story
id: "story-windows-app-settings-window"
epic: "DDC-008"
title: "Windows app: Settings window"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [windows, tauri, app, settings]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Windows app: Settings window

| Field | Value |
|---|---|
| **Epic** | [DDC-008](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The macOS Settings window on Windows ([[adr/windows-app-tauri-2-shell-over-the-ddc-cli|Windows app: Tauri 2 shell over the ddc CLI]]): the same tabs and the same `settings.json` keys, which the app writes atomically itself as on macOS (macOS ADR §6 item 6) (`SettingsModel`, `AppPreferences`, `FolderList`).

- **General:** start at sign-in (the `Run` key through `tauri-plugin-autostart`, shown from the system's state, never a stored bool); show the size next to the icon (the tooltip on Windows, which has no text beside a tray icon); a dot on the tray icon after a notification; delete Review items permanently by default.
- **General** also shows the **Command-line tool** row, as on macOS: where `ddc` is installed and whether it is on `PATH`.
- **Automation:** scan automatically daily or weekly at an hour, or off; pause; notify only over the threshold; only during working hours. It reads the interval, hour and paused state from `schedule status --json` and writes through `ddc schedule` (story "Windows app: scheduled scans and notifications"). App-only preferences live in `%APPDATA%\com.smartandpoint.deepdrivecleaner\preferences.json` (app ADR §4).
- **Folders:** folders the scan skips, and the folders it could not read, from the report's `inaccessible` list (ACL-denied folders exist on Windows although `full_disk_access` is always true there).
- **Types:** include Safe and Rebuildable; hide items smaller than a size.
- **Permissions:** macOS's Full Disk Access tab has no Windows counterpart. The tab is replaced by **About**: version, licence, the CLI's path, and whether notifications can show: the AUMID is registered (ADR §4) and Windows' own setting for the app's notifications is on (`ToastNotifier.Setting`), and the `deepdrivecleaner:` protocol is registered (without it the toast's buttons do nothing).

## Decomposition

- [ ] `ddc-app-model`: settings and preferences models, ported with their tests.
- [ ] The Settings window and its tabs; the context menu's Settings… entry enabled.

## Implementation Plan

## Acceptance Criteria

- [ ] Model tests cover each setting's read, write and the values the CLI refuses.
- [ ] Start at sign-in shows the `Run` key's real state after it is changed outside the app.
- [ ] Manual (owner): each tab's change survives a restart of the app.

## Dependencies

- Story "Windows app: Tauri shell, tray icon and the panel".

## Final Summary

## Attachments

-

---

*Last updated: 2026-11-01*

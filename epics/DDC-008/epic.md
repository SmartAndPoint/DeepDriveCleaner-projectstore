---
type: epic
id: "DDC-008"
title: "Native installers and the Windows tray app"
status: planned
priority: p2
created: 2026-10-31
updated: 2026-11-02
external_refs: {}
tags: [windows, macos, distribution, installer, signing, tray, schedule, tauri, app]
code_refs: []
review_status: pending
reviewed_at: null
---

# DDC-008: Native installers and the Windows tray app

| Field | Value |
|---|---|
| **Status** | planned |
| **Priority** | p2 |
| **Created** | 2026-10-31 |
| **Updated** | 2026-11-01 |

---

## Goal

A person who does not use a terminal can install DeepDriveCleaner the way they install any other app: download one file from the website, open it and follow the installer. Windows comes first (the owner, 2026-10-31).

- **Windows.** A signed installer. When it finishes, the person has:
  - the CLI (`deepdrivecleaner` and `ddc`) on their `PATH`;
  - the Windows app: an icon in the notification area whose panel, Review, Whole disk and Settings windows match the macOS app (full parity, built with Tauri 2: the owner, 2026-11-01). It runs scheduled scans, says what can be freed, and cleans what the person confirms.
  
  The signature is in the name of a verified Windows publisher. Windows shows that publisher's name, not "Unknown publisher".
- **macOS.** A signed, notarized DMG for direct download, next to the Homebrew cask.

Success criteria:

- On a Windows 11 machine, the installer runs:
  - without administrator rights;
  - with the publisher's verified name in its prompts;
  - with only the warnings the distribution ADR predicts.
  
  Afterwards a new terminal runs `ddc --version`, and the app's icon is in the notification area.
- The app does what the macOS app does: the panel with the last scan, Scan now, Review and clean with confirmation (what can be put back goes to the Recycle Bin, as on macOS), Settings, and the Whole disk and coding agents views where the Windows CLI declares them. Each DDC-002 behaviour it covers is a row in a parity table of its story.
- The app schedules scans daily or weekly, or turns them off. After a scheduled scan that finds enough to free, it shows a Windows notification. It deletes nothing on its own: only what a person confirmed in the confirmation window.
- Uninstalling from Settings → Apps leaves no binaries, `PATH` entry, shortcut, sign-in entry or scheduled task behind.
- A DMG downloaded by a browser on a clean Mac opens and installs, and the app starts with no Gatekeeper warning beyond the usual first-launch confirmation.

## Context

- **Scope widened on 2026-11-01.** The owner asked for the Windows app with "a menu similar to macOS", full parity, built with Tauri 2, and put the Windows scanner first. The decision and its alternatives (WinUI 3, Slint, a bare `tray-icon` menu) are in [[adr/windows-app-tauri-2-shell-over-the-ddc-cli|Windows app: Tauri 2 shell over the ddc CLI]]. The minimal tray story became the first of five app stories.
- The macOS app ([[epics/DDC-002/epic|DDC-002 · Menu bar app]]) is a Swift shell over the CLI ([[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]). Its bundled agent runs `DeepDriveCleaner --tick` hourly. The app decides when a scan is due and runs `scan --background --notify json`. It shows the notification and records it with `notify ack` or `notify snooze`. `ddc schedule install` is the CLI-only path and is refused while the app schedules.
- On Windows the scheduler is owned differently. The app delegates scheduling to `ddc schedule`, whose Task Scheduler task runs `DeepDriveCleaner.exe --scheduled`. Task Scheduler handles daily and weekly triggers and missed runs itself, so no due function is needed. The notify decision rule is shared with macOS; the scheduler ownership is not, and the app writes no `scheduler` record.
- The Windows port of the core is [[epics/DDC-007/epic|DDC-007 · Standalone CLI on macOS, Linux and Windows: portable core, release builds and installers]]. Until its story "Windows scanner and rule catalog" flips Windows to supported, the Windows CLI runs only `--version`, `--help` and `rules`. The app's real scans and the scheduler need that story first; the app is built against a fake CLI before it. The signing research and the installer do not. The Whole disk and coding agents views also need DDC-007's stories "Whole disk on Windows: the areas outside the home" and "Coding agents on Windows: ddc disk agents".
- DDC-007's story "Windows install: PowerShell script, winget and a signed binary" planned the Windows signing research and the route ADR. On 2026-10-31 the owner chose a signed installer as the first Windows route. The research and the ADR move to this epic's story "Windows signing and distribution route: research and ADR". That story keeps `install.ps1` and winget, which come after the installer and use the same signed files.
- The DMG story sits here, not in DDC-002, because the owner asked for both installers together. A DMG reverses one line of the menu-bar ADR: "A zip, not a DMG: the cask is the install path". The DMG story amends that line. The cask and the Sparkle feed keep the zip.
- Changes on Microsoft's side since 2024 shape the signing story. These facts were checked on 2026-10-31, and the research note re-verifies them:
  - an EV certificate no longer gives immediate SmartScreen reputation;
  - Microsoft Store registration is free for individuals (since September 2025) and companies (since May 2026), and the Store re-signs an MSIX package itself;
  - Azure Artifact Signing (formerly Trusted Signing) accepts individual developers only in the USA and Canada.
- Signing belongs to the owner, as on macOS. Account names, tenant ids and certificate details stay out of the vault, as the Apple team ID does.

## Stories

| Story | Status | Description |
|-------|--------|-------------|
| [story-windows-signing-and-distribution-route-research-and-adr](./stories/story-windows-signing-and-distribution-route-research-and-adr.md) | planned | Research note on what Windows 11 does with signed and unsigned setups and the signing routes (Store MSIX, Artifact Signing, an OV certificate); the setup format is settled by the app ADR. Then the ADR. The owner picks the publisher identity |
| [story-windows-scheduled-scans-through-task-scheduler](./stories/story-windows-scheduled-scans-through-task-scheduler.md) | planned | `ddc schedule` on Windows: a per-user Task Scheduler task running `DeepDriveCleaner.exe --scheduled`, catch-up after a missed run, an hour, pause and resume, `status --json` with the next run. Declares the `schedule` capability |
| [story-windows-app-tauri-shell-tray-icon-and-the-panel](./stories/story-windows-app-tauri-shell-tray-icon-and-the-panel.md) | planned | The app ADR, `crates/ddc-app-model` ported from `DDCKit`, the Tauri shell in `app-windows/`, a static UI, the tray icon and its states, the panel with the last scan and Scan now, single instance, a fake CLI, the Gate's app job |
| [story-windows-app-review-window-and-clean-with-confirmation](./stories/story-windows-app-review-window-and-clean-with-confirmation.md) | planned | The Review window and the confirmation sheet at parity with DDC-002: groups, sort, selection, agents' sessions, dry-run warnings, Recycle Bin or permanent, held-open skips, the result |
| [story-windows-app-scheduled-scans-and-notifications](./stories/story-windows-app-scheduled-scans-and-notifications.md) | planned | `--scheduled`, the toast with Review and Remind me tomorrow, Pause and Resume, the next-run line |
| [story-windows-app-settings-window](./stories/story-windows-app-settings-window.md) | planned | General, Automation, Folders and Types as on macOS; About in place of Permissions |
| [story-windows-app-whole-disk-and-coding-agents-views](./stories/story-windows-app-whole-disk-and-coding-agents-views.md) | planned | The Whole disk view and the agents section, each shown only when the Windows CLI declares its capability |
| [story-windows-installer-one-signed-setup-with-the-cli-and-the-tray](./stories/story-windows-installer-one-signed-setup-with-the-cli-and-the-tray.md) | planned | One signed per-user setup from Tauri's NSIS bundler: the app, the CLI in `bin\` on `PATH`, a Start menu entry with the AUMID and an Apps entry. Upgrades in place and uninstalls cleanly. Silent switches for winget. Tested in CI |
| [story-windows-download-from-the-website-a-signed-setup-from-a-trusted-publisher](./stories/story-windows-download-from-the-website-a-signed-setup-from-a-trusted-publisher.md) | planned | Download for Windows on the product site and the CLI site (English and Chinese): every release fact read at runtime from the releases API, a plain fallback link, the warnings a person meets and where the publisher shows, verify-yourself commands; reputation builds from downloads under one signing identity |
| [story-macos-a-notarized-dmg-for-direct-download](./stories/story-macos-a-notarized-dmg-for-direct-download.md) | planned | A signed, notarized, stapled DMG with the app, for the website's download button. The cask and Sparkle keep the zip |

Order (the owner, 2026-11-01): DDC-007 "Windows scanner and rule catalog" first; then the app's shell and panel against a fake CLI, the scheduler, Review and clean, notifications, Settings, the installer, the website download (added 2026-11-02); the signing research beside them; the disk and agents views when their CLI stories land.

## Expected Results

- [ ] The Windows distribution ADR names the signing route, the installer format and the exact prompts a user sees.
- [ ] A release carries a signed Windows installer, covered by `SHA256SUMS`, built from the release commit after its Windows gate passed.
- [ ] Windows users get scheduled scans and notifications with the same decision rule as macOS.
- [ ] The Windows app covers each DDC-002 behaviour that has a Windows meaning; each app story's parity table lists them, and the ones without a Windows meaning (Full Disk Access, Homebrew, Sparkle for now) with the reason.
- [ ] A release carries a notarized DMG.

## Dependencies

- DDC-007 story "Windows scanner and rule catalog", for the app's real scans and the scheduler.
- DDC-007 stories "Whole disk on Windows: the areas outside the home" and "Coding agents on Windows: ddc disk agents", for the app's views of them.
- DDC-007 story "Build matrix: every target built, tested and packaged from one tag", for the release jobs that build and sign the installer.
- The product website (`website/` in the code repository), for the download buttons.

## Open Questions

- [ ] The publisher identity on Windows: the owner as an individual, or the SmartAndPoint organization. This decides which signing routes are open. The owner answers; the vault records only the route.
- [ ] Store MSIX, a directly downloaded setup, or both. Decided by the ADR from the research note. An early answer on the publisher identity prunes the research: an individual outside the USA and Canada has an OV certificate or the Store.
- [ ] Whether scheduled scans on Windows are on or off after install. The proposal is off, with an opt-in on the installer's last page. The Mac app is on by default, daily. The owner decides.
- [ ] Updates on Windows. There is no Sparkle counterpart. The proposal: winget upgrade, or downloading the new setup, with an update check in the tray as a later story. Recorded as a non-goal of this epic until the owner says otherwise.

## Related

- [[adr/windows-app-tauri-2-shell-over-the-ddc-cli|Windows app: Tauri 2 shell over the ddc CLI]]

- [[adr/platform-seam-one-platform-module-per-os-with-declared-capabilities|Platform seam: one platform module per OS with declared capabilities]]
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]
- [[research/distribution-through-agents|Distribution through agents]]

---

*Last updated: 2026-11-02*

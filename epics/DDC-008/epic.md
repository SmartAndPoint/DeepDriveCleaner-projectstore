---
type: epic
id: "DDC-008"
title: "Native installers and the Windows tray app"
status: planned
priority: p2
created: 2026-10-31
updated: 2026-10-31
external_refs: {}
tags: [windows, macos, distribution, installer, signing, tray, schedule]
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
| **Updated** | 2026-10-31 |

---

## Goal

A person who does not use a terminal can install DeepDriveCleaner the way they install any other app: download one file from the website, open it and follow the installer. Windows comes first (the owner, 2026-10-31).

- **Windows.** A signed installer. When it finishes, the person has:
  - the CLI (`deepdrivecleaner` and `ddc`) on their `PATH`;
  - a small icon in the notification area that runs scheduled scans and says what can be freed.
  
  The signature is in the name of a verified Windows publisher. Windows shows that publisher's name, not "Unknown publisher".
- **macOS.** A signed, notarized DMG for direct download, next to the Homebrew cask.

Success criteria:

- On a Windows 11 machine, the installer runs:
  - without administrator rights;
  - with the publisher's verified name in its prompts;
  - with only the warnings the distribution ADR predicts.
  
  Afterwards a new terminal runs `ddc --version`, and the tray icon is in the notification area.
- The tray icon schedules scans daily or weekly, or turns them off. After a scheduled scan that finds enough to free, it shows a Windows notification. It never deletes anything.
- Uninstalling from Settings → Apps leaves no binaries, `PATH` entry, shortcut, sign-in entry or scheduled task behind.
- A DMG downloaded by a browser on a clean Mac opens and installs, and the app starts with no Gatekeeper warning beyond the usual first-launch confirmation.

## Context

- The macOS app ([[epics/DDC-002/epic|DDC-002 · Menu bar app]]) is a Swift shell over the CLI ([[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]). Its bundled agent runs `DeepDriveCleaner --tick` hourly. The app decides when a scan is due and runs `scan --background --notify json`. It shows the notification and records it with `notify ack` or `notify snooze`. `ddc schedule install` is the CLI-only path and is refused while the app schedules.
- On Windows the scheduler is owned differently. The tray delegates scheduling to `ddc schedule`, whose Task Scheduler task runs `ddc-tray.exe --scheduled`. Task Scheduler handles daily and weekly triggers and missed runs itself, so no due function is needed. The notify decision rule is shared with macOS; the scheduler ownership is not, and the tray writes no `scheduler` record.
- The Windows port of the core is [[epics/DDC-007/epic|DDC-007 · Standalone CLI on macOS, Linux and Windows: portable core, release builds and installers]]. Until its story "Windows scanner and rule catalog" flips Windows to supported, the Windows CLI runs only `--version`, `--help` and `rules`. The tray and the scheduler need that story first. The signing research and the installer do not.
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
| [story-windows-signing-and-distribution-route-research-and-adr](./stories/story-windows-signing-and-distribution-route-research-and-adr.md) | planned | Research note on what Windows 11 does with signed and unsigned installers, the signing routes (Store MSIX, Artifact Signing, an OV certificate) and the installer formats. Then the ADR. The owner picks the publisher identity |
| [story-windows-scheduled-scans-through-task-scheduler](./stories/story-windows-scheduled-scans-through-task-scheduler.md) | planned | `ddc schedule` on Windows: a per-user Task Scheduler task in place of the LaunchAgent, that catches up a missed run. Declares the `schedule` capability |
| [story-windows-tray-app-a-minimal-scheduling-icon](./stories/story-windows-tray-app-a-minimal-scheduling-icon.md) | planned | `ddc-tray.exe`: icon and tooltip with the last result, Scan now, Schedule (daily, weekly, off), a notification after a scheduled scan with Remind me tomorrow, Start at sign-in, Quit. A shell over the CLI that never deletes |
| [story-windows-installer-one-signed-setup-with-the-cli-and-the-tray](./stories/story-windows-installer-one-signed-setup-with-the-cli-and-the-tray.md) | planned | One signed per-user setup: the CLI on `PATH`, the tray, a Start menu entry and an Apps entry. Upgrades in place and uninstalls cleanly. Silent switches for winget. Tested in CI on `windows-latest` |
| [story-macos-a-notarized-dmg-for-direct-download](./stories/story-macos-a-notarized-dmg-for-direct-download.md) | planned | A signed, notarized, stapled DMG with the app, for the website's download button. The cask and Sparkle keep the zip |

## Expected Results

- [ ] The Windows distribution ADR names the signing route, the installer format and the exact prompts a user sees.
- [ ] A release carries a signed Windows installer, covered by `SHA256SUMS`, built from the release commit after its Windows gate passed.
- [ ] Windows users get scheduled scans and notifications with the same decision rule as macOS.
- [ ] A release carries a notarized DMG.

## Dependencies

- DDC-007 story "Windows scanner and rule catalog", for the tray and the scheduler.
- DDC-007 story "Build matrix: every target built, tested and packaged from one tag", for the release jobs that build and sign the installer.
- The product website (`website/` in the code repository), for the download buttons.

## Open Questions

- [ ] The publisher identity on Windows: the owner as an individual, or the SmartAndPoint organization. This decides which signing routes are open. The owner answers; the vault records only the route.
- [ ] Store MSIX, a directly downloaded setup, or both. Decided by the ADR from the research note. An early answer on the publisher identity prunes the research: an individual outside the USA and Canada has an OV certificate or the Store.
- [ ] Whether scheduled scans on Windows are on or off after install. The proposal is off, with an opt-in on the installer's last page. The Mac app is on by default, daily. The owner decides.
- [ ] Updates on Windows. There is no Sparkle counterpart. The proposal: winget upgrade, or downloading the new setup, with an update check in the tray as a later story. Recorded as a non-goal of this epic until the owner says otherwise.

## Related

- [[adr/platform-seam-one-platform-module-per-os-with-declared-capabilities|Platform seam: one platform module per OS with declared capabilities]]
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]
- [[research/distribution-through-agents|Distribution through agents]]

---

*Last updated: 2026-10-31*

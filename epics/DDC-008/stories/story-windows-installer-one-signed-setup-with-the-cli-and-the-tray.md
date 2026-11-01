---
type: story
id: "story-windows-installer-one-signed-setup-with-the-cli-and-the-tray"
epic: "DDC-008"
title: "Windows installer: one signed setup with the CLI and the tray"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-31
updated: 2026-11-01
external_refs: {}
tags: [windows, installer, distribution, signing, release, ci]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Windows installer: one signed setup with the CLI and the tray

| Field | Value |
|---|---|
| **Epic** | [DDC-008](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

A person downloads one file from the website or the GitHub release, opens it, and gets DeepDriveCleaner (the owner, 2026-10-31). The signing route comes from the ADR of story "Windows signing and distribution route: research and ADR". The setup itself is Tauri's per-user NSIS bundle (amended 2026-11-01, [[adr/windows-app-tauri-2-shell-over-the-ddc-cli|Windows app: Tauri 2 shell over the ddc CLI]] §7), with an `installerHooks` file for what the bundler does not do. If the distribution ADR also picks MSIX, it is a second package from the same binaries, and the same requirements apply through its manifest.

What the installer does:
- **Per user, no administrator rights.** Installs to `%LOCALAPPDATA%\Programs\DeepDriveCleaner` through a custom NSIS template that changes only the `InstallDir` line of Tauri's (app ADR §7): the stock folder, `%LOCALAPPDATA%\DeepDriveCleaner`, is the CLI's data folder. `install.ps1` (DDC-007 story "Windows install: PowerShell script, winget and a signed binary", not written yet) must use the same folder and layout. One install per user, whichever route made it. A second route detects the first and upgrades it in place, or refuses with a message; it never installs a copy beside it.
- **Contents:**
  - the app, `DeepDriveCleaner.exe`, with its UI resources;
  - `bin\deepdrivecleaner.exe` and `bin\ddc.exe`. `ddc.exe` is a second copy of the binary, because a setup cannot make the hard link `install.ps1` makes. The CLI sits in `bin\` because `DeepDriveCleaner.exe` and `deepdrivecleaner.exe` would be one file on Windows (app ADR §1);
  - `LICENSE.md`, `DISCLAIMER.md` and `README.md`, as in the CLI archive.
- **Registers:**
  - the `bin\` folder on the user `PATH` (a checkbox, on by default), never the app's own folder;
  - a Start menu shortcut for the app, carrying its AppUserModelID, without which its notifications do not show (app ADR §4, §6);
  - the `deepdrivecleaner:` URL protocol, through which the toast's buttons reach the app (app ADR §6);
  - "Start at sign-in" for the app (a checkbox, on by default): the `Run` value `DeepDriveCleaner`, the same name and data `tauri-plugin-autostart` writes;
  - an entry in Settings → Apps with the publisher, the version and the uninstaller.
- **The schedule is off by default** (proposal; the owner decides, see the epic's Open Questions). The last page offers "Scan weekly and tell me what can be freed", which runs `ddc schedule install --interval weekly`. Nothing scans without the person's choice. The page is hidden while Windows is unsupported, because `ddc schedule` exits 69 until DDC-007's story "Windows scanner and rule catalog" flips it.
- **Upgrade = run a newer setup.** It closes a running app instance (through its single-instance channel) and replaces the files. A running `bin\deepdrivecleaner.exe` or `bin\ddc.exe` (a `ddc scan` in a terminal) is renamed to `*.old` by the pre-install hook before the copy, because the bundler's own check closes only `DeepDriveCleaner.exe`; the app deletes `*.old` at its next start. After an upgrade, `ddc --version` equals `deepdrivecleaner --version`. It keeps the settings, the schedule and the notify state.
- **Uninstall** removes the files, the `PATH` entry, the shortcut, the sign-in entry and the scheduled task (`ddc schedule uninstall` before the files go). It asks whether to also remove settings and scan history ("Remove my settings and reports"): the CLI's data folder, the app's preferences folder and the WebView2 profile. The default keeps them, as reinstalls expect. A silent uninstall keeps them too, unless it is given a remove-data switch, named in the ADR.
- **Silent switches**, for winget and for CI: an unattended install and uninstall with no prompts, per user.
- **Signed.** The setup and the uninstaller it writes are signed, as are all three executables, through the chosen route and timestamped. `SHA256SUMS` covers the signed setup.
- **Release asset:** `DeepDriveCleaner-<version>-windows-x86_64-setup.exe`. Built only after the Windows gate passed for the release commit. On ARM64 Windows it installs the x86_64 build, which runs under emulation, and the release notes say so.

## Decomposition

- [ ] The bundle configuration in `app-windows/src-tauri/tauri.conf.json` (`installMode: currentUser`, the CLI as resources in `bin\`), the custom template with its test against the upstream template of the pinned Tauri version, a test that the install folder is not the data folder, and the NSIS hooks in `packaging/windows/hooks.nsh`: `PATH`, the protocol, the `Run` value, the rename-aside of `bin\*.exe`, the scheduled task and the remove-data choice on uninstall.
- [ ] `scripts/build-windows-installer.sh` (or a job step in `release.yml`): build the binaries, sign them, build the setup, sign it, checksum.
- [ ] CI in the Gate's Windows job, against a locally built, unsigned setup:
  - a silent install, then `ddc --version` from a new shell whose `PATH` is read from the registry;
  - an upgrade while the app and `ddc.exe` are running, after which `ddc --version` equals `deepdrivecleaner --version`;
  - an install over an `install.ps1` install, after which no `PATH` element and no executable of that install's layout is left;
  - a silent uninstall, then a check that `%LOCALAPPDATA%\Programs\DeepDriveCleaner`, the `Uninstall` key, the `Run` entry, the `PATH` element, the shortcut and the task are gone, and the data folder is still there;
  - a second silent uninstall with the remove-data switch, after which the data folder is gone too.
- [ ] Release integration: the setup joins the release assets and `SHA256SUMS`. The release runbook "Cutting a release: tag, CI archives, dist and publish" gains the Windows signing step.
- [ ] The website's Windows download button (`website/`) links the latest setup.
- [ ] Update DDC-007's story "Windows install: PowerShell script, winget and a signed binary": winget can point at this setup (`InstallerType: nullsoft`, `Scope: user`) instead of a portable zip.

## Implementation Plan

## Acceptance Criteria

- [ ] The CI checks in the decomposition pass on `windows-latest`.
- [ ] The release's setup and the executables it installs carry valid, timestamped signatures from the chosen publisher (`Get-AuthenticodeSignature` in the release job).
- [ ] `SHA256SUMS` lists the setup, and the release job verifies it.
- [ ] Manual (owner, lab machine or VM), recorded in the owner-checks runbook:
  - the setup is downloaded by a browser and installed, and the prompts match the ADR;
  - `ddc --version` works in a new terminal;
  - the app's icon appears in the notification area;
  - the prompts name the verified publisher (the signature). Settings → Apps shows the installer's own publisher text, which is not proof of signing;
  - uninstalling from Settings → Apps leaves nothing behind.

## Final Summary

## Technical Notes

- Changing the user `PATH` needs a broadcast (`WM_SETTINGCHANGE`) so that new terminals see it. Terminals already open do not. The finish page says so.
- Installing over a running app: the bundler's `CheckIfAppIsRunning` closes `DeepDriveCleaner.exe` through the Restart Manager; the CLI's executables are renamed aside by the hook. The CI test covers both.
- The lab machine allows installs only with the owner's go-ahead for that run. Afterwards, record what the run left and remove it (runbook "Native runs: the Linux and Windows legs on the owner's machines").

## Dependencies

- Story "Windows signing and distribution route: research and ADR".
- Story "Windows app: Tauri shell, tray icon and the panel" (for the app; a CLI-only setup can ship first if the owner wants).
- DDC-007 story "Build matrix: every target built, tested and packaged from one tag".

## Attachments

-

---

*Last updated: 2026-11-01*

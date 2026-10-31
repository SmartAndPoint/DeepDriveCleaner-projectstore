---
type: story
id: "story-windows-installer-one-signed-setup-with-the-cli-and-the-tray"
epic: "DDC-008"
title: "Windows installer: one signed setup with the CLI and the tray"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-31
updated: 2026-10-31
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

A person downloads one file from the website or the GitHub release, opens it, and gets DeepDriveCleaner (the owner, 2026-10-31). The format and the signing route come from the ADR of story "Windows signing and distribution route: research and ADR". This story builds what that ADR picks. The text below assumes a setup executable (the working proposal is Inno Setup). If the ADR picks MSIX, the same requirements apply, met through the package manifest.

What the installer does:
- **Per user, no administrator rights.** Installs to `%LOCALAPPDATA%\Programs\DeepDriveCleaner`, the folder `install.ps1` uses. One install per user, whichever route made it. A second route detects the first and upgrades it in place, or refuses with a message; it never installs a copy beside it.
- **Contents:**
  - `deepdrivecleaner.exe` and `ddc.exe`. `ddc.exe` is a second copy of the binary, because a setup cannot make the hard link `install.ps1` makes;
  - `ddc-tray.exe`;
  - `LICENSE.md`, `DISCLAIMER.md` and `README.md`, as in the CLI archive.
- **Registers:**
  - the folder on the user `PATH` (a checkbox, on by default);
  - a Start menu shortcut for the tray app, carrying its AppUserModelID if the tray ADR uses the shortcut for notifications;
  - "Start at sign-in" for the tray (a checkbox, on by default);
  - an entry in Settings → Apps with the publisher, the version and the uninstaller.
- **The schedule is off by default** (proposal; the owner decides, see the epic's Open Questions). The last page offers "Scan weekly and tell me what can be freed", which runs `ddc schedule install --interval weekly`. Nothing scans without the person's choice. The page is hidden while Windows is unsupported, because `ddc schedule` exits 69 until DDC-007's story "Windows scanner and rule catalog" flips it.
- **Upgrade = run a newer setup.** It closes a running tray instance (through its single-instance mutex) and replaces the files. A running `deepdrivecleaner.exe` or `ddc.exe` is renamed aside, not overwritten (Inno Setup's `restartreplace`, or a rename like `install.ps1`'s). After an upgrade, `ddc --version` equals `deepdrivecleaner --version`. It keeps the settings, the schedule and the notify state.
- **Uninstall** removes the files, the `PATH` entry, the shortcut, the sign-in entry and the scheduled task (`ddc schedule uninstall` before the files go). It asks whether to also remove settings and scan history. The default keeps them, as reinstalls expect. A silent uninstall keeps them too, unless it is given a remove-data switch, named in the ADR.
- **Silent switches**, for winget and for CI: an unattended install and uninstall with no prompts, per user.
- **Signed.** The setup and the uninstaller it writes are signed, as are all three executables, through the chosen route and timestamped. `SHA256SUMS` covers the signed setup.
- **Release asset:** `DeepDriveCleaner-<version>-windows-x86_64-setup.exe`. Built only after the Windows gate passed for the release commit. On ARM64 Windows it installs the x86_64 build, which runs under emulation, and the release notes say so.

## Decomposition

- [ ] The installer script in `packaging/windows/`, following the ADR.
- [ ] `scripts/build-windows-installer.sh` (or a job step in `release.yml`): build the binaries, sign them, build the setup, sign it, checksum.
- [ ] CI in the Gate's Windows job, against a locally built, unsigned setup:
  - a silent install, then `ddc --version` from a new shell whose `PATH` is read from the registry;
  - an upgrade while the tray and `ddc.exe` are running, after which `ddc --version` equals `deepdrivecleaner --version`;
  - an install over an `install.ps1` install;
  - a silent uninstall, then a check that `%LOCALAPPDATA%\Programs\DeepDriveCleaner`, the `Uninstall` key, the `Run` entry, the `PATH` element, the shortcut and the task are gone, and the data folder is still there;
  - a second silent uninstall with the remove-data switch, after which the data folder is gone too.
- [ ] Release integration: the setup joins the release assets and `SHA256SUMS`. The release runbook "Cutting a release: tag, CI archives, dist and publish" gains the Windows signing step.
- [ ] The website's Windows download button (`website/`) links the latest setup.
- [ ] Update DDC-007's story "Windows install: PowerShell script, winget and a signed binary": winget can point at this setup (`InstallerType: inno`, `Scope: user`) instead of a portable zip.

## Implementation Plan

## Acceptance Criteria

- [ ] The CI checks in the decomposition pass on `windows-latest`.
- [ ] The release's setup and the executables it installs carry valid, timestamped signatures from the chosen publisher (`Get-AuthenticodeSignature` in the release job).
- [ ] `SHA256SUMS` lists the setup, and the release job verifies it.
- [ ] Manual (owner, lab machine or VM), recorded in the owner-checks runbook:
  - the setup is downloaded by a browser and installed, and the prompts match the ADR;
  - `ddc --version` works in a new terminal;
  - the tray icon appears;
  - the prompts name the verified publisher (the signature). Settings → Apps shows the installer's own publisher text, which is not proof of signing;
  - uninstalling from Settings → Apps leaves nothing behind.

## Final Summary

## Technical Notes

- Changing the user `PATH` needs a broadcast (`WM_SETTINGCHANGE`) so that new terminals see it. Terminals already open do not. The finish page says so.
- Installing over a running tray: Inno Setup's `AppMutex` or the Restart Manager can close it. The ADR picks one; the CI test covers it.
- The lab machine allows installs only with the owner's go-ahead for that run. Afterwards, record what the run left and remove it (runbook "Native runs: the Linux and Windows legs on the owner's machines").

## Dependencies

- Story "Windows signing and distribution route: research and ADR".
- Story "Windows tray app: a minimal scheduling icon" (for the tray parts; a CLI-only setup can ship first if the owner wants).
- DDC-007 story "Build matrix: every target built, tested and packaged from one tag".

## Attachments

-

---

*Last updated: 2026-10-31*

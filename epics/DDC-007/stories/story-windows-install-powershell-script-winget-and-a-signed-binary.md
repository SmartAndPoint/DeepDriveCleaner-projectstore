---
type: story
id: "story-windows-install-powershell-script-winget-and-a-signed-binary"
epic: "DDC-007"
title: "Windows install: PowerShell script, winget and a signed binary"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-31
external_refs: {}
tags: [distribution, install, windows, signing, adr]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Windows install: PowerShell script, winget and a signed binary

| Field | Value |
|---|---|
| **Epic** | [DDC-007](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

On Windows the owner wants something like the macOS and Linux one-liner, "but maybe it needs a signed `.exe` installer; this needs thought" (2026-10-28). The question that decides the outcome is not the installer format. It is whether the binary is signed, because Windows judges the executable, not the way it arrived.

Options to weigh:

| Route | For | Against |
|---|---|---|
| `irm https://ddc.sh/install.ps1 \| iex` + zip with the `.exe` | Like the other platforms; per-user; no admin | An unsigned `.exe` may be blocked by Smart App Control on Windows 11 regardless of how it was downloaded. Needs verification |
| winget, "portable" package (the same zip) | The Windows counterpart of Homebrew: `winget install SmartAndPoint.DeepDriveCleaner` | Manifest reviewed in `microsoft/winget-pkgs`; unsigned binaries are more likely to be flagged |
| Signed MSI installer (`cargo-wix`) | Familiar for managed deployment | Heaviest; no benefit without signing; nobody has asked for managed deployment |

Signing options, to price and check:
- Azure Trusted Signing: a monthly subscription, with eligibility rules for organisations and individuals.
- OV or EV certificates on a hardware token or cloud HSM: hundreds of dollars a year.
- Whether EV still gives immediate SmartScreen reputation: it is believed not to since 2024. Verify; do not assume.

Signing belongs to the owner, as it does on macOS.

## Decomposition

- [ ] Research note "Installing an unsigned and a signed CLI on Windows 11", verified on a real Windows 11 machine or VM:
  - what Smart App Control and SmartScreen do with an unsigned `.exe` downloaded by a browser, by `Invoke-WebRequest`, and by winget;
  - the same for a signed one;
  - what winget's validation does with unsigned portable packages;
  - signing routes with eligibility, price and CI integration (signing in GitHub Actions without exposing keys).
  
  Run the critic on it.
- [ ] ADR "Windows distribution route", from the note. The owner chooses the signing route; until then the ADR records the unsigned route and its warnings.
- [ ] `packaging/install/install.ps1`:
  - `PROCESSOR_ARCHITECTURE` `AMD64` → x86_64; `ARM64` → the x86_64 asset with a note about x64 emulation;
  - latest version via the release redirect;
  - verifies against `SHA256SUMS`;
  - installs to `%LOCALAPPDATA%\Programs\DeepDriveCleaner`;
  - `ddc.exe` as a hard link, recreated on every install and upgrade (rename the old link aside like the binary, then link again): a hard link keeps the old file's data, so a link that is not recreated keeps running the previous version;
  - adds to the user `PATH` only with `-AddToPath`, otherwise prints the command;
  - `-Version`, `-Uninstall`;
  - upgrade = run again: rename the current `deepdrivecleaner.exe` aside to `deepdrivecleaner.exe.old` (a running `.exe` can be renamed but not replaced or deleted), move the new one in, relink `ddc.exe`, delete the `.old` files, or leave it for the next run if it is still in use;
  - the same exit codes as `install.sh`;
  - fails closed;
  - `$env:DDC_INSTALL_BASE_URL` for tests.
- [ ] CI on `windows-latest` against a local fake release: fresh install, upgrade, checksum mismatch, uninstall.
- [ ] Once a signing route is chosen: sign `deepdrivecleaner.exe` in `release.yml` (or locally, as on macOS) before packaging. `SHA256SUMS` covers the signed archive.
- [ ] winget manifest (portable package) submitted to `microsoft/winget-pkgs` by the owner; an update step in the release runbook.
- [ ] Hosting at `https://ddc.sh/install.ps1` through DDC-004's site, as for `install.sh`.

## Implementation Plan

## Acceptance Criteria

- [ ] The research note and the ADR exist, critic-approved; the ADR names the chosen route and, if unsigned, the exact warning a user sees.
- [ ] On a clean Windows runner, `install.ps1` gives working `deepdrivecleaner --version` and `ddc --version`; a rerun with a newer fake release upgrades, and afterwards `ddc --version` equals `deepdrivecleaner --version`; an upgrade while the old binary is running succeeds; a tampered archive installs nothing; `-Uninstall` leaves nothing behind (CI).
- [ ] Manual (owner, Windows 11 machine or VM): the chosen route installs and runs the release with the warnings the ADR predicts and no others.
- [ ] If winget is chosen: `winget install SmartAndPoint.DeepDriveCleaner` installs the release (manual, owner).

## Final Summary

## Technical Notes

- The owner's Windows machines (2026-10-31) build and test the code through `scripts/native-windows.sh`. See runbook "Native runs: the Linux and Windows legs on the owner's machines". The build machine is not an install test bed: the owner allows nothing outside its `ddc-native` folder. An install or winget check of `ddc` may run on the lab machine with the owner's go-ahead for that run; record what it left (a PATH entry, an Apps entry, a winget package) and remove it after. Smart App Control is on only after a clean Windows 11 install, so its check runs in a VM; whether Windows Sandbox has it is unverified. SmartScreen's verdict depends on Mark-of-the-Web and the file's reputation, not on a clean machine. The machines are private: nothing that describes them goes into the note.
- `irm | iex` runs the script text without saving a file, so the execution policy does not apply to the script itself. The downloaded `.exe` is a different matter, which is why the binary's signature decides the result.

## Dependencies

- Story "Windows scanner and rule catalog" (the Windows archive exists and is supported).
- Story "Build matrix: every target built, tested and packaged from one tag".
- DDC-004 story "English and Chinese product website with development runbook" (hosting).

## Attachments

-

---

*Last updated: 2026-10-31*

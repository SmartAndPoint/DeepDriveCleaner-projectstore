---
type: runbook
slug: "native-runs-the-linux-and-windows-legs-on-the-owners-machines"
title: "Native runs: the Linux and Windows legs on the owner's machines"
status: draft
date: 2026-10-31
authors: ["Evgenii Konev"]
tags: [linux, windows, ci, testing, ddc-007]
---

# Native runs: the Linux and Windows legs on the owner's machines

## Purpose

The Gate runs the workspace natively on Linux and Windows in its `check-<target>` jobs. A CI round for one fix takes tens of minutes. The owner has a Linux machine and two Windows machines reachable over ssh, and two scripts run the same leg on them in minutes:

- `scripts/native-linux.sh` for Linux;
- `scripts/native-windows.sh` for Windows.

Use them for every Linux or Windows change of epic DDC-007, in particular stories "Linux scanner and rule catalog", "Windows scanner and rule catalog" and "Build matrix: every target built, tested and packaged from one tag". Iterate there, and push once a whole native run is green. CI stays the last check, not the loop: the `check-<target>` jobs remain required on `main`.

**The machines are private.** The code repository and this vault are public, and nothing that describes these machines goes into either. That covers host names, addresses, user names, OS builds, hardware, what else runs on them, and measurements that describe them. It applies to commit messages, pull request bodies and release notes as well. Write "a Linux machine with docker" or "a Windows machine". Counts are fine: tests run, tests skipped, the skip list's length. Anything that describes the machine is not: run times, core counts, disk sizes, its filesystem, its accounts, or what else is installed on it. One carve-out: fixtures this project makes on the lab machine for the scanner may be listed by path relative to the home, and what the scanner finds in them may be reported. What was on the machine before us may not be listed. The scripts take the host only from the environment (`DDC_LINUX_HOST`, `DDC_WINDOWS_HOST`) and have no default. The ssh alias lives only in the developer machine's `~/.ssh/config`.

## Prerequisites

- [ ] An ssh alias for each machine in the developer machine's `~/.ssh/config`, and key login that works without a prompt: `ssh -o BatchMode=yes <alias> true`.
- [ ] **Linux machine**:
  - docker, with the ssh user in the `docker` group;
  - curl. Then, once and again after either pin changes, from the developer machine:

    ```bash
    DDC_LINUX_HOST=<alias> scripts/native-linux.sh --setup
    ```

    This installs rustup (with `--no-modify-path`), the toolchain of `rust-toolchain.toml` and the cargo-nextest `.config/tools.mk` pins inside `~/ddc-native`, for the machine's architecture. The host user's own `~/.rustup` and `~/.cargo` are never used or changed (0.14.4; before it, runs used them).
  - No sudo and no C compiler on the host are needed: the tests run in `rust:<toolchain>-bookworm`.
  - Both architectures are worth a machine. CI runs clippy and the tests on x86_64 Linux only and builds aarch64 for the release without clippy; an aarch64 run caught a clippy error that only aarch64 has (`st_nlink` is 32 bits there).
  - The target folder grows with every version bump. Runs build without incremental data, as CI does, and print a warning past 40 GiB. They never remove it: what to delete on the owner's machine is the owner's call.
  - **Or WSL on a Windows machine** (code 0.14.7). Set `DDC_LINUX_WSL=<distribution>` beside `DDC_LINUX_HOST=<the Windows alias>`, and every command goes through `wsl.exe` into that distribution, from its home. Git bash's path conversion is off (`MSYS_NO_PATHCONV=1`). The distribution needs:
    - a C toolchain (`build-essential`, `pkg-config`);
    - docker, with the user in the `docker` group;
    - systemd, so docker starts with the distribution.

    Installing these needs `sudo` inside WSL, which is the owner's to give. Run `--setup` once, as on a Linux machine. If ssh stops answering after the WSL install, Windows may have turned the network Public, and the OpenSSH firewall rule covered Private only. The fix is the owner's: `Set-NetConnectionProfile -InterfaceIndex <n> -NetworkCategory Private`, or the rule widened to `-Profile Any -RemoteAddress LocalSubnet`.
- [ ] **Each Windows machine**, prepared once by its owner. Copy `scripts/windows-host-setup.ps1` to the machine, open PowerShell as Administrator and run:

  ```powershell
  Set-ExecutionPolicy -Scope Process Bypass -Force
  .\windows-host-setup.ps1 -PubKey '<the developer machine's ssh public key>'
  ```

  The script needs an administrator. It changes these things on the machine, and Rollback undoes them:
  - installs OpenSSH Server, starts it at every boot and restarts it at the end of each run;
  - adds a firewall rule for port 22 on the Private profile, if no OpenSSH rule exists yet;
  - writes the key to `%USERPROFILE%\.ssh\authorized_keys` and to `C:\ProgramData\ssh\administrators_authorized_keys`, and limits the latter to Administrators and SYSTEM;
  - makes Git bash the ssh shell (`HKLM\SOFTWARE\OpenSSH\DefaultShell`);
  - installs Git for Windows and Visual Studio Build Tools with the C++ workload;
  - turns on long paths (`LongPathsEnabled` in the registry, `core.longpaths` in Git's system config);
  - creates the work folder in the home (`-Dir`, default `ddc-native`) and excludes it from Defender's scan.

  `native-windows.sh` reads the same folder name from `DDC_WINDOWS_DIR` (default `ddc-native`). If one of the two is changed, change the other to match.

  Then, from the developer machine:

  ```bash
  DDC_WINDOWS_HOST=<alias> scripts/native-windows.sh --setup
  ```

  This installs rustup (with `--no-modify-path`), the pinned toolchain and the pinned nextest inside `ddc-native`. Run it again after either pin changes. rustup's current release writes a Settings → Apps entry (`HKCU\…\Uninstall\Rustup`) only when it also edits PATH. Its development branch writes the entry regardless. Either way, `--setup` removes the entry when it points into the folder.

## Steps

### 1. Run the whole leg

```bash
DDC_LINUX_HOST=<alias> scripts/native-linux.sh
DDC_WINDOWS_HOST=<alias> scripts/native-windows.sh
```

Each script does the same things:

1. Copies the working tree's tracked files to the machine. Files git no longer tracks are deleted there. **Untracked files are not copied**: `git add` a new test before running, or it will not run. The script names any untracked files it left behind.
2. Runs what the CI job runs:
   - `cargo clippy --workspace --all-targets --locked -- -D warnings`;
   - `cargo nextest run --workspace --locked` with `NEXTEST_PROFILE=ci`;
   - the doc tests;
   - on Windows, all three with `--target x86_64-pc-windows-msvc`, as the CI job passes it;
   - `scripts/native-skips.sh <os>` on the test log.

A run takes minutes, where a CI round takes tens of minutes.

### 2. Run a few tests while fixing one

```bash
DDC_WINDOWS_HOST=<alias> scripts/native-windows.sh -E 'test(junction)'
```

With nextest filter arguments, a script runs only the matching tests. It skips clippy, the doc tests and the skip compare, so a green filtered run says nothing about those. Finish with a whole run (step 1) before pushing.

### 3. Push

Push only after a whole native run is green. The CI job on the pull request is the final check.

## The Windows machines: the build in one folder

The owner sets a rule per Windows machine (2026-10-31):

- **A build machine.** Everything stays inside `ddc-native` in the home. Nothing goes on the machine's PATH or anywhere else.
- **A lab machine.** The build stays inside `ddc-native` the same way. Outside it, the machine may also hold test projects, tools and environments (Python, Node and the like), so the Windows scanner has a developer's disk to find. Make them with a purpose and keep them in a few known places. The story that made them lists them by path relative to the home, as the carve-out above allows. Use the lab machine for Windows development by default. An install script or winget check of `ddc` there still needs the owner's go-ahead for that run.

Which alias is which machine cannot be written in either repo. It is recorded only on the developer machine, as a comment beside each `Host` in `~/.ssh/config`. Read that comment before any run of `ddc` against a real home.

On both machines the build itself lives here:

| Folder | Holds | Set by |
|---|---|---|
| `ddc-native/src` | the synced tracked files | `native-windows.sh` |
| `ddc-native/target` | the build, and `native-tests.log` | `CARGO_TARGET_DIR` |
| `ddc-native/cargo`, `ddc-native/rustup` | rustup, the toolchain, nextest, the registry | `CARGO_HOME`, `RUSTUP_HOME`, `--setup` |
| `ddc-native/home` | the tests' home: `HOME`, `USERPROFILE`, `HOMEDRIVE` and `HOMEPATH`, `APPDATA` (`home\AppData\Roaming`) and `LOCALAPPDATA` (`home\AppData\Local`) | `native-windows.sh` |
| `ddc-native/tmp` | `TMP` and `TEMP` | `native-windows.sh` |

Every variable a Windows program reads its home from points inside the folder. A test, or the port's own folders code, that writes to its home never reaches the machine's real profile. When the Windows port decides where the home and data folders come from, it must read them from these variables. If it calls an API that ignores them, the profile's known-folder lookup for example, a test would write to the real profile.

OpenSSH gives an administrator's session an elevated token, as CI's runner has. An elevated run is not what a person's unelevated session meets: for example, it has the symlink privilege.

`TMP` and `TEMP` sit beside the tests' `HOME`, never inside it, as on the CI runner: the test trees' guard refuses a temp folder inside the home.

On a build machine, never run `ddc` against the real home: only `--version`, `--help` and `rules` are allowed until the owner says otherwise. Installing anything else there is the owner's call. That includes an install script under test and a research measurement of the real home. On a lab machine, scanning and measuring its home are fine. A real clean there needs the owner's go-ahead each time.

## Verification

- [ ] The run ends with exit 0, and the last line reads `native-skips: <n> tests skip on <os>, as listed (...)`.
- [ ] The same commit's `check-<target>` job in CI passes. A difference between the two is a finding: record it in the story.
- [ ] Before pushing, check the diff for the machine's alias, host name and address:

  ```bash
  git diff origin/main | grep -i -E '<alias>|<host name>|<address>'
  ```

  The command must print nothing.

## Rollback

The scripts write nothing on the developer machine. On the lab machine, first remove the fixtures the stories list. Then, to remove everything from either Windows machine:

1. Delete `ddc-native`. Settings → Apps may still list a Rustup whose `UninstallString` points into `ddc-native`. If it does, delete `HKCU\Software\Microsoft\Windows\CurrentVersion\Uninstall\Rustup`.
2. At the machine's own console, not over ssh (this step ends ssh), in PowerShell as administrator:

   ```powershell
   Remove-MpPreference -ExclusionPath $env:USERPROFILE\ddc-native
   Remove-NetFirewallRule -Name OpenSSH-Server-In-TCP
   Stop-Service sshd
   Remove-WindowsCapability -Online -Name (Get-WindowsCapability -Online -Name 'OpenSSH.Server*').Name
   Remove-Item -Recurse -Force HKLM:\SOFTWARE\OpenSSH
   Remove-Item -Recurse -Force C:\ProgramData\ssh
   ```

   The last line removes sshd's host keys and the administrators' key file with it.
3. Remove the developer machine's key line from `%USERPROFILE%\.ssh\authorized_keys`.
4. If long paths were off before the setup, run `Set-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem LongPathsEnabled 0` and `git config --system --unset core.longpaths`.
5. Uninstall Visual Studio Build Tools and Git through Settings → Apps, if nothing else needs them.

On the Linux machine, delete `~/ddc-native` and the image: `docker rmi rust:<toolchain>-bookworm`.

## Common Issues

### `ssh: connect to host … port 22: Operation timed out` (Windows)

The network is marked Public, and the OpenSSH firewall rule covers only the Private profile. That holds for the rule the setup script adds, and it can hold for the rule Windows created with the capability. If it is a home network, mark it Private as administrator:

```powershell
Set-NetConnectionProfile -InterfaceAlias '<Wi-Fi or Ethernet name>' -NetworkCategory Private
```

To check, run `Get-NetConnectionProfile` and `Get-NetFirewallRule -Name OpenSSH-Server-In-TCP | ft Profile`.

### `Permission denied (publickey)` (Windows)

For an administrator account, sshd reads keys only from `C:\ProgramData\ssh\administrators_authorized_keys`, and only when Administrators and SYSTEM alone can access that file. The setup script writes the key there and sets that access. If a key was added by hand, rerun the script.

### `native-skips: …` lists a difference

A test skipped that the list does not name, or a listed test no longer skips. Either update `.config/native-skips/<os>.txt` in the same change, or fix the test. The skip list is a reviewed diff (ADR "Platform seam: one platform module per OS with declared capabilities", §9).

### Most of the synced tree vanished on the machine

The cause was a locale mismatch in the delete step. Both sides must sort with `LC_ALL=C`, and the scripts set it. Do not drop it from a copy of the script.

### Results differ when the Linux container runs as root

root reads folders the tests make unreadable, so tests that expect a refusal pass where they should fail. `native-linux.sh` runs the container as the ssh user's uid. Never run it as root.

## References

- `scripts/native-linux.sh`, `scripts/native-windows.sh`, `scripts/windows-host-setup.ps1`, `scripts/native-skips.sh`, `.config/native-skips/`.
- `.github/workflows/gate.yml`, job `check`.
- `AGENTS.md` in the code repository, section "Linux and Windows on the owner's machines".
- Story "Windows scanner and rule catalog" (DDC-007).
- Story "Linux scanner and rule catalog" (DDC-007).

---

*Last updated: 2026-10-31*

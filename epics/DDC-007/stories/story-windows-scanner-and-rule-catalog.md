---
type: story
id: "story-windows-scanner-and-rule-catalog"
epic: "DDC-007"
title: "Windows scanner and rule catalog"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-31
external_refs: {}
tags: [rust, windows, scanner, rules, safety]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Windows scanner and rule catalog

| Field | Value |
|---|---|
| **Epic** | [DDC-007](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

This story fills `platform::windows` and the Windows catalog section. Windows differs from Unix in ways that bear directly on safety:
- some reparse points (symlinks, junctions, mount points) are links, and following one into a deletion would delete the target;
- there are no `*at` calls, so the safety ADR's verified-parent deletes must be rebuilt on file handles;
- paths are case-insensitive and may exceed `MAX_PATH`;
- a file opened without `FILE_SHARE_DELETE` cannot be deleted.

Each of these is a rule with a test.

As for Linux, a research note comes first: where dev disk space goes on Windows, with measured sizes.

## Decomposition

- [ ] Research note "Where a developer's disk space goes (Windows)". Measure:
  - `%LOCALAPPDATA%` and `%APPDATA%` tool caches (npm-cache, pip, uv, NuGet `%USERPROFILE%\.nuget\packages`, gradle, maven, cargo, go, JetBrains, VS Code);
  - `%TEMP%`;
  - Visual Studio caches;
  - WSL distributions' `ext4.vhdx` and Docker Desktop's disk image (VM class: reported, never deleted).
  
  Run the critic on it.
- [ ] Walker: `GetFileInformationByHandleEx(FileIdExtdDirectoryInfo)` on the directory handle. It returns, per entry and in batches, the name, attributes, `AllocationSize`, `EndOfFile`, 128-bit `FileId` and `ReparsePointTag`. This is the analogue of `getattrlistbulk`. Fall back to `FindFirstFileExW` (what `std::fs::read_dir` uses, plus `FIND_FIRST_EX_LARGE_FETCH`) only where the volume does not support it (FAT, exFAT). Measure both on a large tree.
- [ ] Hard links: link count via `FileStandardInfo` only for entries whose file id repeats within the scan.
- [ ] Reparse points: an entry is a link when `IsReparseTagNameSurrogate(tag)` (symlink, junction, mount point). Links are not descended and are reported as links. Other tags are files or folders with the tag recorded: cloud placeholders, WOF-compressed files, ProjFS, AppExecLink. Cloud-sync roots (OneDrive) are not walked, as macOS treats `Library/CloudStorage`.
- [ ] One filesystem: the volume serial from the directory handle; mount points are already links.
- [ ] Verified-parent operations: open the parent by handle, check the child's `FileIdInfo` (volume serial + 128-bit id) against the scan, rename to a tombstone with `SetFileInformationByHandle(FileRenameInfo)` relative to the parent handle, re-check the id, then delete with `FILE_DISPOSITION_INFO_EX` (POSIX semantics where supported).
- [ ] Command runner: native cleanup commands run in a Job Object, so a timeout kills the whole tree.
- [ ] Home and paths: `Env::from_process` takes the home from `USERPROFILE` (fallback `HOMEDRIVE` + `HOMEPATH`). Canonical paths drop the `\\?\` prefix for output and comparison (as the `dunce` crate does), while Win32 calls get the verbatim form. Comparisons in the catalog and protected-folder checks are case-insensitive, and run on the path from `GetFinalPathNameByHandleW` of an open handle, never on the string the user typed: 8.3 short names (`PROGRA~1`) and per-directory case sensitivity (`fsutil file setCaseSensitiveInfo`) would otherwise let a protected folder slip through.
- [ ] Files in use: a sharing violation at delete time is reported as "held open" and skipped, never retried in a loop. Open-file checks before a clean use the Restart Manager API where it answers, otherwise the capability is stated as absent.
- [ ] Recycle Bin via the `trash` crate.
- [ ] Catalog (Windows section): rules from the research note. Protected folders: `%USERPROFILE%\.ssh`, `%APPDATA%\Microsoft\Protect` (DPAPI master keys: deleting them is irrecoverable), `%APPDATA%\Microsoft\Crypto`, `%APPDATA%\Microsoft\Credentials`, `%LOCALAPPDATA%\Microsoft\Credentials`, `%LOCALAPPDATA%\Microsoft\Windows\INetCookies`, browser profiles, every folder protected by kind, and the system folders the shared section from story "Rule catalog split: shared and per-OS sections" leaves empty (drive roots, `Windows`, `Program Files`, `Program Files (x86)`, `ProgramData`, `Users`, `%APPDATA%`, `%LOCALAPPDATA%`), amended into the safety ADR §6 before `SUPPORTED` flips. The catalog split's flip-guard test compares home-relative paths (`protected_in_home` joins them to the home); since `%APPDATA%` can be redirected, this story decides whether it also needs an `Env`-based protected set. It also decides whether `Env` carries `APPDATA` and `TEMP`, which the catalog split handed over from safety ADR §8.
- [ ] Data and logs: `%LOCALAPPDATA%\DeepDriveCleaner`, through `env::Env`.
- [ ] Non-UTF-8 names: Windows names can hold unpaired surrogates (WTF-8). Represent them the way the schema does for other non-UTF-8 names.
- [ ] Honour the "when absent" column of ADR "Platform seam: one platform module per OS with declared capabilities", §3, for every capability this platform does not declare: `notifications` (`scan --background` decides but does not notify), `full-disk-access` (`full_disk_access` is true), `private-size` (no `private_bytes`) and `local-snapshots` (`local_snapshots` is 0). Story "Platform seam: ADR and capability model" left them unimplemented: no supported platform lacked one. If `in-use-check` is not declared, the flip commit also amends the safety ADR §5 to name the sharing-violation guard that replaces it (seam ADR §3).
- [ ] Flip `SUPPORTED` for Windows and add the target to `release.yml`'s enabled list, in one commit.

## Implementation Plan

## Acceptance Criteria

- [ ] The research note exists, critic-approved, with measured sizes.
- [ ] A junction pointing outside the scanned tree is reported as a link, not descended, and nothing behind it is ever planned (test).
- [ ] A folder swapped for a junction between scan and clean is skipped as changed since the scan, not followed (test).
- [ ] A WOF-compressed or cloud-placeholder file is scanned as a file, not as a link (test, where the runner supports it).
- [ ] A path longer than 260 characters is scanned and cleaned (test).
- [ ] Deleting a file held open without `FILE_SHARE_DELETE` by a test process is reported as held open and skipped; the clean completes (test).
- [ ] The CLI starts with `HOME` unset and reports home paths without the `\\?\` prefix (test).
- [ ] On a clean Windows machine (manual, owner or VM), `ddc scan ~ --json` finds the catalog's caches and validates against the schema.
- [ ] A pre-release tag publishes the Windows archive.

## Final Summary

## Technical Notes

- `GetCompressedFileSizeW` is not the allocated size: it returns the on-disk size only for compressed or sparse files, and the logical size otherwise. `AllocationSize` from the directory information class is the comparable number.
- `unsafe` for Win32 calls stays inside `platform::windows`, each block with a `// SAFETY:` comment (enforced by DDC-006's clippy story).
- Test debt from stage 2 of story "Build matrix: every target built, tested and packaged from one tag" (2026-10-31). `.config/native-skips/windows.txt` held 417 names when stage 2 merged, and it grows until the flip. The flip should leave only the skips for capabilities Windows does not declare. Beyond the walker and `fsops`:
  - The disclaimer tests (`disclaimer_cli.rs`) need a ConPTY driver; Windows has no `script`.
  - `exec::system` `waiting_for_a_tool_does_not_spin` spawns `/bin/sleep` literally.
  - Some tests feed macOS-shaped absolute paths, which Windows does not read as absolute, and need Windows-shaped inputs: `a_malformed_list_is_refused_before_anything_runs`, `agents::running` (`codex_is_a_cli_or_its_app_server`, `cursor_is_its_main_binary_only`, `cursor_agent_is_cursors_cli_by_its_install_folder`), `agents::liveness` `a_harness_runs_by_its_process_or_a_held_codex_lock`, `free::probe` (two tests), `keep` `only_an_absolute_path_without_parent_steps_is_plain`, and the fuzz regressions' "at least one accepted" half.
  - Fixtures that make symlinks skip without the privilege (`common::try_symlink`): `disk_free.rs` `last_disk_*` and `settings.rs`. `there_is_no_yes` (`disk_free_cli.rs`) failed on Windows on its fixture's symlink too, and skips through `unsupported()`.
  - `no_system_process_owns_a_cache` reads the live process table, which is unported, and checks `/System`, `/usr` and app-bundle paths. It needs a Windows meaning, or a reason to stay macOS-only. On a developer machine that takes Developer Mode; the CI runner (`windows-2025`) has no such privilege.
  - The native job's Tests step sets `TMP` and `TEMP` to `runner.temp`. Under bash the temp folder lies inside `HOME`, and the test trees' guard refuses that.
- **Iterate on the owner's Windows machines, not through CI rounds** (the owner, 2026-10-31). `DDC_WINDOWS_HOST=<alias> scripts/native-windows.sh` runs the Gate's Windows leg there in minutes. It runs clippy, nextest with profile `ci`, the doc tests and `scripts/native-skips.sh windows`, for `x86_64-pc-windows-msvc` as the CI job does. With filter arguments it runs only the matching tests. Its runs on `main` at 0.13.1 matched CI: every test passed, and the skips equalled `.config/native-skips/windows.txt`. Each step's pull request is pushed after a green whole native run, and CI's `check-x86_64-pc-windows-msvc` is the last check. Setup, the per-machine rules and failures are in runbook "Native runs: the Linux and Windows legs on the owner's machines".
  - The machines are private. Nothing that describes them goes into this vault or the code repository: no host, address, user, OS build, hardware or run times. The research note's numbers likewise state findings about the tools, not about the machine.
  - The owner has two Windows machines with different rules (2026-10-31). On the build machine everything stays inside `ddc-native`. The lab machine, the default for this story, may also hold test projects, tools and environments (Python, Node and the like) outside that folder. The research note measures the lab machine's home, and the scanner is tried against it. The fixtures made for it are listed here by path relative to the home, and the note's numbers are findings about those fixtures and the tools, not about the machine. A real clean there needs the owner's go-ahead each time. The tests still use the folder's own home on both.
  - `native-windows.sh` points `HOME`, `USERPROFILE`, `HOMEDRIVE` and `HOMEPATH`, `APPDATA` and `LOCALAPPDATA` into the folder. The "Home and paths" and "Data and logs" items must take the home and the data folder from these variables. If one uses an API that ignores them, such as the known-folder lookup, a test run there writes into the owner's real profile, and no test catches it. `crates/ddc-cli/tests/structure.rs` already confines reading `HOME`, `USERPROFILE`, `HOMEDRIVE`, `HOMEPATH` and `LOCALAPPDATA` to the platform's `folders` code. `APPDATA` is not on that list yet.
  - OpenSSH gives an administrator's session an elevated token, as CI's runner has, so an elevated run has the symlink privilege. A test of what an unelevated person meets needs an unelevated run, and this setup does not give one.

## Dependencies

- Stories "Move OS calls behind the platform module" and "Rule catalog split: shared and per-OS sections".
- Story "Build matrix: every target built, tested and packaged from one tag" (stage 2).

## Attachments

-

---

*Last updated: 2026-10-31*

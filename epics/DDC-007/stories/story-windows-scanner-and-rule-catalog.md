---
type: story
id: "story-windows-scanner-and-rule-catalog"
epic: "DDC-007"
title: "Windows scanner and rule catalog"
status: in-progress
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-11-01
external_refs: {}
tags: [rust, windows, scanner, rules, safety]
code_refs: []
specs: []
started_at: "2026-11-01T19:00:00.000Z"
closed_at: null
plan_updated_at: "2026-11-01T19:00:00.000Z"
---

# Windows scanner and rule catalog

| Field | Value |
|---|---|
| **Epic** | [DDC-007](../epic.md) |
| **Status** | in-progress |
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
- [ ] Catalog (Windows section): rules from the research note. Verify on a Windows home the Windows paths of the shared rules story "Linux scanner and rule catalog" added (code PRs #44, #48): JetBrains, the VS Code-family caches and workspace storage, Docker Desktop, the browsers, Spotify, Slack and the coding agents. Protected folders: `%USERPROFILE%\.ssh`, `%APPDATA%\Microsoft\Protect` (DPAPI master keys: deleting them is irrecoverable), `%APPDATA%\Microsoft\Crypto`, `%APPDATA%\Microsoft\Credentials`, `%LOCALAPPDATA%\Microsoft\Credentials`, `%LOCALAPPDATA%\Microsoft\Windows\INetCookies`, browser profiles, every folder protected by kind, and the system folders the shared section from story "Rule catalog split: shared and per-OS sections" leaves empty (drive roots, `Windows`, `Program Files`, `Program Files (x86)`, `ProgramData`, `Users`, `%APPDATA%`, `%LOCALAPPDATA%`), amended into the safety ADR §6 before `SUPPORTED` flips. The catalog split's flip-guard test compares home-relative paths (`protected_in_home` joins them to the home); since `%APPDATA%` can be redirected, this story decides whether it also needs an `Env`-based protected set. It also decides whether `Env` carries `APPDATA` and `TEMP`, which the catalog split handed over from safety ADR §8.
- [ ] Data and logs: `%LOCALAPPDATA%\DeepDriveCleaner`, through `env::Env`.
- [ ] Non-UTF-8 names: Windows names can hold unpaired surrogates (WTF-8). Represent them the way the schema does for other non-UTF-8 names.
- [ ] Honour the "when absent" column of ADR "Platform seam: one platform module per OS with declared capabilities", §3, for every capability this platform does not declare: `notifications` (`scan --background` decides but does not notify), `full-disk-access` (`full_disk_access` is true), `private-size` (no `private_bytes`) and `local-snapshots` (`local_snapshots` is 0). Story "Platform seam: ADR and capability model" left them unimplemented: no supported platform lacked one. If `in-use-check` is not declared, the flip commit also amends the safety ADR §5 to name the sharing-violation guard that replaces it (seam ADR §3).
- [ ] Flip `SUPPORTED` for Windows and add the target to `release.yml`'s enabled list, in one commit.

## Implementation Plan

Planned 2026-11-01 with the planner, from origin/main b70a437 (0.16.0); revised the same day after the critic (verdict REVISE: the Recycle Bin and held-open blockers below). The owner put this story first, before the Windows app of DDC-008. Each step is its own pull request, green on every required check, iterated on the owner's Windows machine with `scripts/native-windows.sh` before CI, with `SUPPORTED` false until the last step. `require_supported` stands in front of every walk and every delete (`walk_with_reader`, `scan_inner`, `preflight`, `execute`, `free::run`), so no step before the flip can remove a file through the CLI. As for Linux, skips leave `.config/native-skips/windows.txt` only by re-gating on a primitive or by new unit tests in `platform/windows/`, and each step proves itself with a dry flip that is never committed.

**Findings that shape the steps:**
- `MetadataExt::volume_serial_number`, `file_index` and `number_of_links` are unstable (`windows_by_handle`), and `Metadata` has no allocation size, so the pinned `meta::file_id`, `link_count` and `alloc_bytes(&Metadata)` have no Windows body. Step 3 replaces them with path-based pins first (it shrinks if the pinned toolchain has stabilised them).
- The flip-guard's `may_be_supported` and the `has_root` assertions on the Windows lists (`catalog/mod.rs`) judge a drive root with `Path::has_root`, which is false for `C:\` on macOS and Linux.
- 26 CLI tests set only `HOME`; step 1 fixed it with `ClearEnv::home_env`.
- `procs::is_root()` must not mean "elevated" (OpenSSH and the CI runner are elevated); it means LocalSystem only. The owner check that stops an elevated session from cleaning another user's folder is step 7.
- `canonicalize()` returns `\\?\C:\…`. A missed site leaves `\\?\` on one side of a `starts_with`, so an `--exclude`, a skip or a root silently never matches and the folder is planned. The catalog maps, `preflight.rs` and the roots compare components exactly; `guard.rs` already folds case with `to_lowercase`, which changes the length of some Unicode names. Step 2 gives one canonicalization and one key for all of them.
- **Held-open is stage-based.** A folder cannot be renamed while a file under it is open: the rename fails with access denied (5), not a sharing violation. Opening a folder with `DELETE` fails with 32 when it is some process's working folder. A running image refuses deletion with 5 too. So 32 or 5 means "held open" or "denied by ACL" depending on the stage (step 8).
- **The Recycle Bin can delete permanently.** The `trash` crate calls the shell with `FOF_NO_UI | FOF_ALLOWUNDO | FOF_WANTNUKEWARNING`; with no UI the shell destroys an item the bin cannot take (a volume without a bin, an item over the bin's size) instead of failing (step 10).
- `IO_REPARSE_TAG_MOUNT_POINT` is both a junction and a volume mount point. pnpm's `node_modules` packages are junctions and must stay cleanable; a mounted volume inside a finding must make it incomplete, as on macOS (step 6).

**Minimum Windows:** Windows 10 22H2 and Windows 11, x86_64 (POSIX rename and delete need 1709, unprivileged symlinks 1703, nested jobs 8). Below it the CLI still starts and the unsupported calls fail closed. `aarch64-pc-windows-msvc` is a later story; the x86_64 build runs under emulation.

**Gate before code (vault, step 0):** research note "Where a developer's disk space goes (Windows)" with measured sizes; amendments of the seam ADR (§2 `platform/windows/` with `unsafe` through `windows-sys`; §3 Windows declares no capability, the Restart Manager is not used; §4 a 128-bit file id is refused, in the walker and in the guard, never truncated; §8 `APPDATA` on the allowlist) and of the safety ADR (§5: the stage-based held-open guard and the executable and command-line check that replace `in-use-check`, the Recycle Bin checks of step 10, the owner check of elevated sessions, `SeBackupPrivilege` never enabled, a bounded retry at the tombstone rename for antivirus holds; §6: the Windows protected list of step 12). All critic-approved. Steps 2 and 3 are shared refactors that do not depend on these decisions and may start beside them.

1. **Foundation** (done, PR #75, 0.16.3): `platform/windows/` with `errno`, `folders` (`USERPROFILE`, never `HOME` or the known-folder API), `rlimit`, `term`, `testing`; `windows-sys`; `ClearEnv::home_env`. `trash` joins in step 10.
2. **Canonical paths and one path key** (shared, no change on Unix): `fsops::canonicalize` strips `\\?\` and `\\?\UNC\` except for names a non-verbatim path cannot spell (a trailing dot or space, a reserved device name), which keep the prefix and fail closed at the identity check; `fsops::path_key` normalises separators and upper-cases per UTF-16 unit on Windows (identity on Unix) and replaces `guard::fold`. All `canonicalize()` sites move to the pin and clippy forbids the std ones. Tests: the Windows key as a pure function on every OS (case and both separators fold to one key); on Windows only, a mixed-case path finds a protected folder, and an `--exclude` and a skip given in another case or as an 8.3 name still exclude (step 14, under `unsupported()`). The Unix hot path stays allocation-free.
3. **Path-based stat pins** (shared): `meta::lstat`/`stat` returning one `FileStat`; every caller of the three `&Metadata` pins migrates, loadgen included.
4. **Windows `meta`:** handles from `CreateFileW` (`FILE_FLAG_BACKUP_SEMANTICS`, `FILE_FLAG_OPEN_REPARSE_POINT`); identity from `FileIdInfo` (its 64-bit volume serial is the one `dev`, also for step 7's volumes); a 128-bit id is refused; a link only for a name-surrogate tag.
5. **Windows `fsops`:** `DirFd` on handles, children opened relative to the parent (`NtCreateFile`); `stat_child` reports a name-surrogate reparse point as a link, so a folder swapped for a junction is refused as changed (AC3's mechanism); rename in place with `FileRenameInfoEx`, falling back to `FileRenameInfo` on `ERROR_INVALID_PARAMETER`; delete with `FileDispositionInfoEx`, falling back likewise; NTSTATUS through `RtlNtStatusToDosError`. Tree deletion stays std's `remove_dir_all` (handle-based, never follows links, POSIX delete with its own bounded retry); read-only files on FAT and exFAT get their attribute cleared, as `make_dirs_writable` does for folders.
6. **Walker** (`windows/bulk.rs`): `FileIdExtdDirectoryInfo` in 64 KiB batches, the folder opened with share-delete; `FileFullDirectoryInfo` on FAT and exFAT. A mount-point tag is read once (`FSCTL_GET_REPARSE_POINT`): a `\??\Volume{…}` target is another file system (the finding is incomplete), a path target is a link. Cloud placeholders are recognised by their attributes (`RECALL_ON_DATA_ACCESS`, `RECALL_ON_OPEN`, `OFFLINE`) and never opened; every file this module opens (the `open_nofollow_nonblock` pin and a read pin the matchers' marker reads move to) passes `FILE_FLAG_OPEN_NO_RECALL`, so a dehydrated file fails to read instead of downloading. **Hard links:** every regular file with a non-zero allocation is recorded in the walk's deferred `InodeTable` (`scan/inodes.rs`, which already resolves after the walk), and only records seen twice or more are opened for their link count. A file whose other link lies outside the scan roots is seen once and counted as freed (on macOS and Linux it is marked incomplete); the ADR amendment states it. Measured against `bench/load-limits.tsv` on the lab fixtures (pnpm and uv make hard links the norm); if over budget, `nlink = 1` with the overstatement stated in the research note and the ADR. Tests: a junction outside the tree is a link (AC2, unit half); a junction inside a finding is cleaned with it; a WOF-compressed file and a placeholder are files (AC4); a name over 260 characters (AC5, scan half); an unpaired surrogate is one `lossy_paths` hit.
7. **Volumes, usage and the owner check:** `GetDiskFreeSpaceExW`, `GetVolumeInformationW`, mount points; process times and peak working set; `machine_model` is `None`. When the token is elevated, a scan root whose owner SID (`GetNamedSecurityInfoW`) is not the token's user is refused, as root is refused on Unix. The CI runner is elevated, so a test there checks that the user's own root passes; a root owned by another SID is a lab check in the owner-checks runbook.
8. **Held-open guard:** `SkipReason::HeldOpen`, a clean-schema bump (DDCKit with it). The mapping lives in `DirFd::rename_child`: opening the child with `DELETE` fails 32 → held open; the open succeeds but the rename fails 5 → held open (an open file below it); the open fails 5 → failed (ACL). A 5 from `remove_dir_all` after a successful rename is a late holder: failed, with the tombstone's path, as today. Before it is coded, one lab experiment records the matrix (working folder, open file, running executable, read-only attribute) in this step and the ADR amendment. No sleep loop; std's bounded retry stands, plus three attempts 50 ms apart at the rename for antivirus scans. AC6.
9. **Processes, signals and the command runner:** the user's processes from a Toolhelp snapshot, with executable and command line; native commands run one at a time, each in its own Job Object, started with `CREATE_SUSPENDED` and resumed after `AssignProcessToJobObject` (through a thread snapshot, std exposes no thread handle); `kill_group` is `TerminateJobObject`; a normal exit closes the job without killing what it left, as on Unix. `.exe`/`.cmd`/`.bat` lookup without `PATHEXT`; `.cmd` shims (`npm`, `pnpm`, `yarn`) go through std's `cmd.exe` quoting, which refuses unsafe arguments. Ctrl-C through `SetConsoleCtrlHandler`. A hidden `--cancel-event <name>`, for the Windows app (ADR "Windows app: Tauri 2 shell over the ddc CLI" §1), lands here only once that ADR is accepted; until then this step ships the console handler alone. Its meaning is the SIGTERM path: stop after the current item, end the running native command's Job Object, write the journal and the report. Any process of the session can open a named event, so at worst it stops a clean early, which is safe.
10. **Recycle Bin** through the `trash` crate, with checks around it: before the move, the item's volume has a Recycle Bin (`SHQueryRecycleBinW`, which gives the bin's current size) and its allocated size fits that bin's maximum (`MaxCapacity` under `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\BitBucket\Volume\{<volume GUID>}`, in MB); after it, the source is gone and the item is listed in the bin. Otherwise it is skipped with a reason that names `--permanent`, or, when the shell destroyed it, reported as removed and not recycled. A path the shell refuses (longer than `MAX_PATH`) is skipped, never deleted in its place; AC5's "cleaned" holds for `delete` items. Tested with a small bin quota on the test home.
11. **Redirected folders:** `APPDATA` and `LOCALAPPDATA` outside the home are protected through `Env`; `APPDATA` joins the environment allowlist.
12. **Protected folders and the flip guard:** the protected list in the home (`AppData` and its three children, `Microsoft\Protect`, `Crypto`, `Credentials`, `Vault`, `SystemCertificates`, `INetCookies`, the browser profiles, `source` (Visual Studio's `~\source\repos`) beside `Projects`, `%APPDATA%\gnupg`, `%LOCALAPPDATA%\Packages`, `%LOCALAPPDATA%\wsl`, `OneDrive`, `Videos`, `Favorites`, `Saved Games`) and the system folders (drive roots, `Windows`, `Program Files`, `Program Files (x86)`, `ProgramData`, `Users`); the flip guard's root test and the lists' `has_root` assertions fixed for `X:\` on every OS.
13. **Catalog rules:** the Windows section's own list from the research note (NuGet, `%TEMP%`'s contents with its running-program caveat, Visual Studio caches, the WSL and Docker Desktop disks as report-only VM items), cloud roots not walked in this story: OneDrive's roots identified by the `OneDrive`, `OneDriveConsumer` and `OneDriveCommercial` variables (business folders are named `OneDrive - <organisation>` and can be moved, so a fixed name is not enough; the allowlist grows in step 11), and Dropbox's and Box's default folders listed (walking them with placeholders recognised, and declaring `dataless-files`, is a later story); the Recycle Bin's per-user folder `<drive>:\$Recycle.Bin\<SID>` is a Whole disk area, not a home finding; every shared Windows path checked on the lab home.
14. **Tests reshaped, and the flip's CLI tests:** Windows-shaped inputs for the path tests; the ConPTY disclaimer tests, `no_system_process_owns_a_cache` and the capability skips stay skipped with a reason; the CLI tests for AC2, AC3, AC5 and AC7 (no output path carries `\\?\`) land here under `unsupported()`, proved by a dry flip; check which tests need `TEMP`/`TMP`, which `clear_env` drops.
15. **The flip, one commit:** `SUPPORTED = true`, `CAPABILITIES = &[]`, the Windows target in `release.yml`, the skip list burnt down to the capability skips, the ADR amendments in force. `crates/ddc-cli/src/platform/unported.rs` keeps serving Windows (a quiet `init_process`), or a `windows.rs` replaces it; the step says which. AC8 goes to the owner-checks runbook; AC9 is met by the next tag.

Open decisions for step 0: hard links (step 6), OneDrive and other cloud roots, `HOME` ignored on Windows, the held-open matrix (step 8, by experiment), per-directory case sensitivity (two names that fold to one key are both protected; a rule anchored at one could claim the other: accepted with a note, or refused).

Proposed `code_refs`: `crates/ddc-core/src/platform/windows/`, `platform/mod.rs`, `platform/types.rs`, `platform/unported.rs`, `rules/catalog/windows.rs`, `rules/catalog/mod.rs`, `rules/mod.rs`, `exec/`, `scan/`, `env.rs`, `crates/ddc-cli/src/platform/`, `crates/ddc-cli/tests/structure.rs`, `crates/ddc-cli/tests/common/`, `crates/ddc-loadgen/src/isolate.rs`, `.config/native-skips/windows.txt`, `.github/workflows/release.yml`, `docs/clean.schema.json`.

## Acceptance Criteria

- [ ] The research note exists, critic-approved, with measured sizes.
- [ ] A junction pointing outside the scanned tree is reported as a link, not descended, and nothing behind it is ever planned (test).
- [ ] A folder swapped for a junction between scan and clean is skipped as changed since the scan, not followed (test).
- [ ] A WOF-compressed or cloud-placeholder file is scanned as a file, not as a link (test, where the runner supports it).
- [ ] A path longer than 260 characters is scanned and cleaned (test).
- [ ] Deleting a file held open without `FILE_SHARE_DELETE` by a test process, and a folder that is a test process's working folder, are reported as held open and skipped; the clean completes (test).
- [ ] A `review` item the Recycle Bin cannot take is skipped or reported as not recycled, never silently destroyed (test with a small bin quota).
- [ ] The CLI starts with `HOME` unset and reports home paths without the `\\?\` prefix (test).
- [ ] On a clean Windows machine (manual, owner or VM, recorded in the owner-checks runbook), `ddc scan ~ --json` finds the catalog's caches and validates against the schema.
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

*Last updated: 2026-11-02*

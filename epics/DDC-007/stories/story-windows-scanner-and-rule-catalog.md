---
type: story
id: "story-windows-scanner-and-rule-catalog"
epic: "DDC-007"
title: "Windows scanner and rule catalog"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
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
- [ ] Catalog (Windows section): rules from the research note. Protected folders: `%USERPROFILE%\.ssh`, `%APPDATA%\Microsoft\Protect` (DPAPI master keys: deleting them is irrecoverable), `%APPDATA%\Microsoft\Crypto`, `%APPDATA%\Microsoft\Credentials`, `%LOCALAPPDATA%\Microsoft\Credentials`, `%LOCALAPPDATA%\Microsoft\Windows\INetCookies`, browser profiles, and every folder protected by kind.
- [ ] Data and logs: `%LOCALAPPDATA%\DeepDriveCleaner`, through `env::Env`.
- [ ] Non-UTF-8 names: Windows names can hold unpaired surrogates (WTF-8). Represent them the way the schema does for other non-UTF-8 names.
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

## Dependencies

- Stories "Move OS calls behind the platform module" and "Rule catalog split: shared and per-OS sections".
- Story "Build matrix: every target built, tested and packaged from one tag" (stage 2).

## Attachments

-

---

*Last updated: 2026-10-28*

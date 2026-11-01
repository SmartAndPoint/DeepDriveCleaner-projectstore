---
type: story
id: "story-linux-file-kinds-special-files-immutable-flags-sparse-files-and-odd-names"
epic: "DDC-010"
title: "Linux file kinds: special files, immutable flags, sparse files and odd names"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [linux, scanner, safety, clean, filesystems]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Linux file kinds: special files, immutable flags, sparse files and odd names

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The Linux walk and clean were built and tested on regular files, folders, symlinks, hard links and names that are not UTF-8. A developer's home holds more than that. This story makes each remaining kind behave as safely as on macOS, with a test where one can run.

- **FIFOs, sockets and device nodes.** These show up in a finding: a dev server's socket in a project's `tmp/`, a FIFO in a build folder, a node inside a container's root filesystem under rootless storage.
  - The walk must count them without opening them.
  - The in-use check must not stall on them.
  - A clean must never read them. The `trash` crate's copy fallback uses `std::fs::copy` on every regular-looking entry when a move crosses filesystems. Copying a FIFO blocks forever, and copying a device node reads the device. Story "Linux mount boundaries: automounts, bind mounts and the Trash across filesystems" removes that fallback for every file, so it is never reached. This story tests that special files are safe on both paths.
- **Immutable and append-only files** (`chattr +i`, `chattr +a`).
  - The rename of such a file fails with `EPERM`. A folder that holds one can be renamed to a tombstone, but its removal then fails, and the user cannot remove the tombstone either.
  - macOS has the same trap with `chflags uchg`, and its tests (`common::user_immutable`) skip on Linux, because only root can set the Linux flag.
  - `statx` reports both flags in `stx_attributes` at no extra cost.
- **Sparse files** (VM disks, database files, `fallocate` holes). The size the scan reports is the allocated size (`stx_blocks`), as `du` counts it. The apparent size can be far larger, and the two must never be mixed up.
- **Names.**
  - A name holding a newline, a tab, a backslash or a leading dash must survive the text report and the suggested `ddc clean …` command (`render::clean_command`).
  - A name of 255 bytes, and a path longer than Linux's 4096-byte `PATH_MAX`, must work. The walk works through descriptors, but the `trash` crate takes paths.
  - Linux filesystems mostly compare names byte for byte. `Cache` and `cache` are two folders there, and a name in NFC is not its NFD twin. The exceptions are folders with ext4's casefold attribute (`chattr +F`, used by Steam and Proton), exFAT and NTFS disks, and CIFS mounts, which are case-insensitive. Rules and protected folders must match the way each filesystem does.
- **Compressed files.** On btrfs with compression and on ZFS, `st_blocks` may not be what a delete frees. Story "btrfs and ZFS on Linux: subvolumes, datasets, snapshots and stable identities" measures this.

## Decomposition

- [ ] Special files.
  - A test tree with a FIFO and a Unix socket inside a matched cache. The device node comes from the CI mounts job, which prepares it with `sudo mknod` and passes its path in a variable.
  - The walk counts each one as 0 bytes and never opens it.
  - A clean to the Trash on the same filesystem renames them. A permanent clean unlinks them.
  - A cross-filesystem Trash move either uses that filesystem's `.Trash-$uid` or refuses, per the mount story. A test proves that no special file is ever opened.
- [ ] Immutable and append-only.
  - Read `STATX_ATTR_IMMUTABLE` and `STATX_ATTR_APPEND` in the walk (`platform/linux/bulk.rs`) and keep them in `EntryMeta`.
  - A finding with such an entry is listed with a new reason, `immutable`, and never planned. The guard's re-walk refuses it before any rename, as it refuses `foreign-owner`.
  - The reason goes into the scan and clean schemas, the docs and DDCKit, with the version moves.
  - Tests use a fake reader on every OS.
  - The real flag: `chattr +i` needs root, and a test cannot set it on a file it creates later. So the CI mounts job prepares an immutable file and an append-only file inside a finding-shaped tree with `sudo chattr`, and passes their paths in variables. The test scans and cleans that tree.
  - `common::user_immutable` gains a Linux branch that uses the prepared tree, and skips with the reason elsewhere.
- [ ] Sparse files: a test with a 1 GiB sparse file that holds 4 KiB of data. The scan, `ddc du` and the clean report all say 4 KiB.
- [ ] Names.
  - Tests for a newline, a tab, a backslash, a leading dash and a 255-byte name: in the JSON, the text report and the suggested clean command. The command, when pasted into `sh`, names the same path. The cases table checks this through its follow-through.
  - A finding whose path is longer than `PATH_MAX`: the scan lists it, and the clean either removes it through descriptors or skips it with the reason `path-too-long`. It never crashes and never leaves a tombstone.
- [ ] Case and Unicode.
  - Tests that `Node_Modules` is not `node_modules` on ext4, that `Cache` and `cache` side by side are two entries, and that a protected folder named in NFC does not protect, or fail to protect, its NFD twin. On macOS the tests state what APFS does instead.
  - On the CI mounts job's casefold ext4 and exFAT, a protected folder written in another case is still protected, and a rule name in another case matches as the filesystem resolves it.
  - How the walk knows. `statx` does not expose casefold, and `f_type` is plain ext4. The walk holds each folder's descriptor, so one `ioctl(FS_IOC_GETFLAGS)` per folder reads `FS_CASEFOLD_FL`, and also `FS_IMMUTABLE_FL` and `FS_APPEND_FL` for the folder itself. For exFAT, NTFS and CIFS, `f_type` is enough.
- [ ] Removal never crosses a mount. `exec/delete.rs` removes a tombstone with `std::fs::remove_dir_all` by path, and `make_dirs_writable` walks by path, so a mount made inside a finding after the guard's check would be descended into. Remove through descriptors with `RESOLVE_NO_XDEV` (from story "Linux mount boundaries: automounts, bind mounts and the Trash across filesystems"), and stop at a mount root.
- [ ] Amend the safety ADR §5 with the special-file and immutable rules, then run the critic on the amendment.

## Implementation Plan

## Acceptance Criteria

- [ ] A FIFO, a socket and a device node inside a finding are counted, never opened, and never copied by a clean (tests; the device node in the CI mounts job).
- [ ] A finding that holds an immutable or append-only file is never planned. A clean of a stale plan that includes one refuses it before the rename and leaves no tombstone (fake-reader test on every OS, plus the prepared `chattr +i` tree in the CI mounts job).
- [ ] On casefold ext4 and exFAT, a protected folder in another case stays protected (test in the CI mounts job).
- [ ] A sparse file is reported at its allocated size by `scan`, `du` and `clean` (test).
- [ ] Names with a newline, a tab, a backslash, a leading dash, 255 bytes, or a path past `PATH_MAX` go through scan, JSON, text and clean without a crash. The suggested clean command names the same path (tests and cases).
- [ ] The safety ADR §5 states the Linux rules for special files and immutable flags, critic-approved.

## Final Summary

## Technical Notes

- Setting and clearing `chattr +i` needs `CAP_LINUX_IMMUTABLE`. The prepared file stays on the CI runner, which is discarded after the job. In native runs the test skips in the skip list's environment group.
- `stx_attributes_mask` says which flags the filesystem reports. tmpfs and NFS report none. There, a rename that fails with `EPERM` is the signal, and it is caught before the removal.
- The new reasons `immutable` and `path-too-long` and their schema version move belong to this story.

## Dependencies

- Story "Linux CI mounts: a filesystem matrix job with root-only steps".
- Story "Linux mount boundaries: automounts, bind mounts and the Trash across filesystems", for the cross-filesystem path.

## Attachments

-

---

*Last updated: 2026-11-01*

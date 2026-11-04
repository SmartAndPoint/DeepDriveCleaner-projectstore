---
type: story
id: "story-linux-file-kinds-special-files-immutable-flags-sparse-files-and-odd-names"
epic: "DDC-010"
title: "Linux file kinds: special files, immutable flags, sparse files and odd names"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-04
external_refs: {}
tags: [linux, scanner, safety, clean, filesystems]
code_refs: []
specs: []
started_at: "2026-11-04T11:05:10.552Z"
closed_at: null
plan_updated_at: "2026-11-04T11:05:10.552Z"
---

# Linux file kinds: special files, immutable flags, sparse files and odd names

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | in-progress |
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

*Superseded on 2026-11-04 by the Implementation Plan and the safety ADR's amendment of that date where they differ: no `FS_IOC_GETFLAGS`, no `path-too-long`, no `trash` crate on Linux, no Linux branch of `common::user_immutable` (a prepared tree instead), and compression moved out.*

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

Planned 2026-11-04 with the planner, from origin/main 6a25d00 (0.30.2); main is now 0e0277f (0.31.0), which moved `model.rs`, `rules/listing.rs` and `store.rs` but no schema version. The contract is the safety ADR's amendment of 2026-11-04 for this story, critic-reviewed.

Already in place from the stories "Linux mount boundaries: automounts, bind mounts and the Trash across filesystems" and "btrfs and ZFS on Linux: subvolumes, datasets, snapshots and stable identities":
- the walk only stats (`statx` with `NO_AUTOMOUNT`), and special files are `EntryKind::Other` at their allocated size;
- the in-use check opens fd links as `O_PATH`;
- the Linux Trash only renames, and there is no `trash` crate on Linux;
- `alloc = stx_blocks × 512`;
- a path past `PATH_MAX` makes the finding `incomplete`;
- the CI mounts job has a device node, single `+i`/`+a` files, a casefold ext4 and an exFAT.

Four code PRs, each with its own bump, merged only after the `linux-mounts` job itself is green:

1. **Special files, sparse, names, exFAT/FAT Trash (patch, no schema move).**
   - `read_found` and both `open_read` open markers with `O_NOFOLLOW | O_NONBLOCK | O_NOCTTY` and `fstat` that they are regular. A symlinked marker no longer counts; test a symlinked worktree `.git` file.
   - `exec/delete.rs` cuts the tombstone name on a character boundary and retries shorter on `ENAMETOOLONG`.
   - `render::clean_command`:
     - keeps a newline or a tab literal inside its single quotes (dash 0.5.12 on Ubuntu 24.04 has no `$'…'`);
     - suggests no command for any other control character or a non-UTF-8 byte.
   - `platform/linux/trash.rs` `fstat`s again after its `fchmod`, checks `files/` and `info/`, and refuses as `trash-crosses-filesystem` when a mode stayed open. Where no `.Trash-$uid` exists yet, the dry run predicts the refusal on exFAT and FAT by `f_type`.
   - Tests:
     - a FIFO and a socket in a finding (trash and permanent);
     - a device-node tree from the CI mounts job, with inotify `IN_OPEN` proving nothing opens it;
     - a 1 GiB sparse file with 4 KiB of data through `scan`, `du` and `clean`;
     - names through a real `/bin/sh`;
     - a 255-byte name removed;
     - cases rows;
     - the exFAT Trash refusal on `DDC_MOUNT_EXFAT`.
2. **Case and Unicode (small).**
   - `.ddc-keep` is compared case-insensitively in `rules/visitor.rs`, `keep_markers` and the guard's `walk_item`.
   - Tests:
     - protected folders in another case on casefold ext4 and exFAT;
     - `Cache`/`cache` and NFC/NFD as two entries on ext4, with APFS stated;
     - rule names byte for byte.
3. **Immutable and append-only (minor, schema move).**
   - The flags come from `statx` in `bulk.rs` (`stat`, `entry_meta`, `stat_child`) and from `st_flags` on macOS.
   - A folder's own flags reach the visitors, through a field on `DirCtx` or a hook.
   - The walk and `--items` get a counter and `NotSelected::Immutable`.
   - Before each action:
     - the guard's per-item re-walk (`walk_item`, the one a fake reader reaches) refuses with `SkipReason::Immutable`;
     - the guard checks the root's own flags;
     - `keep_markers` refuses early.
   - Schemas, DDCKit and wording change. Goldens move by the version line, re-recorded on the Mac.
   - `interrupted_deletion_leaves_a_tombstone_found_as_safe` is rewritten on a hand-made tombstone and leaves both skip lists.
   - Tests:
     - a fake reader through `scan_with_reader` and `walk_item`;
     - a CI `chattr` tree (a file `+i`, a file `+a`, a folder `+i`, a finding root `+i`) cleaned through `--items`.
4. **Removal through descriptors (patch).**
   - A new `DirFd::remove_tree` in each platform's `fsops.rs`, pinned in `platform/mod.rs` and stubbed in `unported.rs`.
   - `exec/delete.rs` leaves paths, `make_dirs_writable` included.
   - A `structure.rs` test checks that `delete.rs` is its only caller.
   - Tests:
     - a unit test of `remove_tree` on a tombstone tree that the CI root step prepares, with a tmpfs, a bind mount and a user-made btrfs subvolume inside. The mounts survive, and the error maps to `failed` naming the tombstone;
     - `interrupt_cli` and loadgen's hostile trees stay green.

## Acceptance Criteria

- [ ] A FIFO, a socket and a device node inside a finding are counted, never opened, and on Linux never copied by a clean. A marker file that is a device, a FIFO, or a symlink to one, reads as absent (tests; the device node in the CI mounts job).
- [ ] A finding that holds an immutable or append-only file is never planned. A clean of a stale plan that includes one refuses it before the rename and leaves no tombstone (fake-reader test on every OS, plus the prepared `chattr` tree in the CI mounts job, cleaned through `--items`).
- [ ] On casefold ext4 and exFAT, a protected folder in another case stays protected. Rule names match byte for byte on every filesystem (tests in the CI mounts job).
- [ ] A sparse file is reported at its allocated size by `scan`, `du` and `clean` (test).
- [ ] Names with a newline, a tab, a backslash, a leading dash or 255 bytes, and a path past `PATH_MAX`, go through scan, JSON, text and clean without a crash. The suggested clean command, pasted into `sh`, names the same path, or no command is suggested (tests and cases).
- [ ] Removing a tombstone with a tmpfs, a bind mount or a btrfs subvolume inside leaves them intact, and the error reports the item `failed`, naming the tombstone (`remove_tree` test on a CI-prepared tree).
- [ ] On exFAT, a `.Trash-$uid` whose mode stays open refuses the move as `trash-crosses-filesystem`, in the dry run too (CI mounts test).
- [ ] The safety ADR §5 and §6 state the rules for special files, immutable flags, removal through descriptors, case, and exFAT/FAT, critic-approved.

## Final Summary

## Technical Notes

- Setting and clearing `chattr +i` needs `CAP_LINUX_IMMUTABLE`. The prepared file stays on the CI runner, which is discarded after the job. In native runs the test skips in the skip list's environment group.
- `stx_attributes_mask` says which flags the filesystem reports. Where it lacks them, the flag reads as not set. A rename of a folder that holds an immutable file succeeds, and `EPERM` arrives only on the unlink, after a partial removal; the ADR accepts that, and the next scan finds the tombstone. Whether tmpfs reports the bits on the CI kernel is checked there.
- No `path-too-long` reason: `incomplete` already refuses a finding deeper than `PATH_MAX`. The new reason `immutable` and its schema version move belong to this story.
- The `trash` crate no longer runs on Linux, and the compression item moved to the story "Compressed btrfs: say when a scan counts more than a clean frees".

## Dependencies

- Story "Linux CI mounts: a filesystem matrix job with root-only steps".
- Story "Linux mount boundaries: automounts, bind mounts and the Trash across filesystems", for the cross-filesystem path.

## Attachments

-

---

*Last updated: 2026-11-04*

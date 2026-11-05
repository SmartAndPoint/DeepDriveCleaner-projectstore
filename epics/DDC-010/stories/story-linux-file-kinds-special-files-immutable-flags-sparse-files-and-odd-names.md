---
type: story
id: "story-linux-file-kinds-special-files-immutable-flags-sparse-files-and-odd-names"
epic: "DDC-010"
title: "Linux file kinds: special files, immutable flags, sparse files and odd names"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-05
external_refs: {}
tags: [linux, scanner, safety, clean, filesystems]
code_refs: [".config/native-skips/linux.txt", ".config/native-skips/windows.txt", "scripts/ci-mounts.sh", "crates/ddc-cli/src/cmd/background.rs", "crates/ddc-cli/src/notify.rs", "crates/ddc-cli/src/render.rs", "crates/ddc-cli/src/prompt.rs", "crates/ddc-cli/tests/cases.json", "crates/ddc-cli/tests/cases_cli.rs", "crates/ddc-cli/tests/clean_cli.rs", "crates/ddc-cli/tests/structure.rs", "crates/ddc-core/src/exec/delete.rs", "crates/ddc-core/src/exec/guard.rs", "crates/ddc-core/src/exec/mod.rs", "crates/ddc-core/src/exec/preflight.rs", "crates/ddc-core/src/keep.rs", "crates/ddc-core/src/model.rs", "crates/ddc-core/src/platform/mod.rs", "crates/ddc-core/src/platform/linux/bulk.rs", "crates/ddc-core/src/platform/linux/fsops.rs", "crates/ddc-core/src/platform/linux/trash.rs", "crates/ddc-core/src/platform/macos/fsops.rs", "crates/ddc-core/src/platform/windows/fsops.rs", "crates/ddc-core/src/report/mod.rs", "crates/ddc-core/src/report/contents.rs", "crates/ddc-core/src/rules/entries.rs", "crates/ddc-core/src/rules/mod.rs", "crates/ddc-core/src/rules/toolchain.rs", "crates/ddc-core/src/rules/visitor.rs", "crates/ddc-core/src/scan/reader.rs", "crates/ddc-core/src/scan/walker.rs", "crates/ddc-core/src/scan/mod.rs", "app/Packages/DDCKit/Sources/DDCKit/ScanReport.swift", "app/Packages/DDCKit/Sources/DDCKit/CleanReport.swift", "app/Packages/DDCKit/Sources/DDCKit/CleanWording.swift", "crates/ddc-core/tests/file_kinds.rs", "crates/ddc-core/tests/clean_safety.rs", "crates/ddc-core/tests/clean_items.rs", "crates/ddc-core/tests/protection.rs", "crates/ddc-core/tests/scanner.rs", "crates/ddc-core/tests/user_data.rs", "crates/ddc-core/tests/keep.rs", "crates/ddc-core/tests/common/mounts.rs", "crates/ddc-core/tests/common/fakes.rs"]
specs: []
started_at: "2026-11-04T11:05:10.552Z"
closed_at: "2026-11-05T08:20:07.866Z"
plan_updated_at: "2026-11-04T11:05:10.552Z"
---

# Linux file kinds: special files, immutable flags, sparse files and odd names

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | done |
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

- [x] A FIFO, a socket and a device node inside a finding are counted, never opened, and on Linux never copied by a clean. A marker file that is a device, a FIFO, or a symlink to one, reads as absent (tests; the device node in the CI mounts job). — evidence: `crates/ddc-core/tests/file_kinds.rs` a_fifo_and_a_socket_in_a_finding_are_never_opened, a_device_node_in_a_finding_is_counted_never_opened_and_unlinked, a_device_node_moves_into_the_trash_unopened (inotify `IN_OPEN`, CI linux-mounts); `clean_safety.rs` a_fifo_where_a_marker_should_be_never_blocks_the_scan; `mounts.rs` the_prepared_device_is_a_character_device (a device reads as no marker); `worktrees.rs` a_worktree_whose_dot_git_is_a_symlink_is_not_one; PR #117 (0.33.1)
- [x] A finding that holds an immutable or append-only file is never planned. A clean of a stale plan that includes one refuses it before the rename and leaves no tombstone (fake-reader test on every OS, plus the prepared `chattr` tree in the CI mounts job, cleaned through `--items`). — evidence: `clean_safety.rs` a_flagged_entry_keeps_a_finding_unselected_and_unremoved, `clean_items.rs` a_flag_set_inside_an_item_after_the_review_is_refused_by_the_dry_run, `user_data.rs` a_picked_folder_with_a_flagged_file_inside_is_refused_by_the_dry_run (fake reader; macOS and Linux, skipped on Windows with their unsupported fixtures, where no such flag exists); `clean_safety.rs` a_locked_file_or_folder_keeps_its_finding_out_of_a_plan_and_a_clean (real `chflags uchg`, macOS gate PASS); `file_kinds.rs` findings_holding_a_flag_only_root_sets_are_listed_and_refused_through_items (CI `chattr` tree, stale plan and `--items`, linux-mounts PASS); PR #124
- [x] On casefold ext4 and exFAT, a protected folder in another case stays protected. Rule names match byte for byte on every filesystem (tests in the CI mounts job). — evidence: `file_kinds.rs` a_protected_folder_in_another_case_is_refused_on_casefold / _on_exfat / _refused, a_rule_name_in_another_case_is_not_the_rule_on_casefold / _on_exfat; `protection.rs` a_rule_name_in_another_case_is_not_the_rule (APFS, ext4); PR #120 (0.34.1)
- [x] A sparse file is reported at its allocated size by `scan`, `du` and `clean` (test). — evidence: `crates/ddc-cli/tests/clean_cli.rs` a_sparse_file_counts_its_allocated_size_in_scan_du_and_clean; PR #117
- [x] Names with a newline, a tab, a backslash, a leading dash or 255 bytes, and a path past `PATH_MAX`, go through scan, JSON, text and clean without a crash. The suggested clean command, pasted into `sh`, names the same path, or no command is suggested (tests and cases). — evidence: `crates/ddc-cli/src/render.rs` tests the_clean_command_pasted_into_sh_names_the_same_paths, control_characters_are_escaped; `crates/ddc-cli/tests/cases.json` (eight `"unix": true` rows) via `cases_cli.rs`; `clean_safety.rs` a_tombstone_with_a_255_byte_name_is_removed; `exec/delete.rs` a_long_name_is_cut_on_a_character_so_the_suffix_fits, a_name_too_long_for_the_filesystem_is_tried_shorter_each_time; PR #117
- [x] Removing a tombstone with a tmpfs, a bind mount or a btrfs subvolume inside leaves them intact, and the error reports the item `failed`, naming the tombstone (`remove_tree` test on a CI-prepared tree). — evidence: `file_kinds.rs` a_removal_never_crosses_a_mount_made_inside_a_tombstone (CI tmpfs, bind mount, user-made btrfs subvolume; linux-mounts PASS); `clean_safety.rs` a_folder_closed_after_the_plan_stops_the_removal_and_keeps_the_tombstone (the `failed` outcome naming the tombstone), a_tombstone_swapped_after_the_check_is_left_whole, a_symlink_inside_a_deleted_finding_is_unlinked_never_followed; `ddc-cli/tests/structure.rs` only_the_tombstone_removal_removes_a_tree; PR #125
- [x] On exFAT, a `.Trash-$uid` whose mode stays open refuses the move as `trash-crosses-filesystem`, in the dry run too (CI mounts test). — evidence: `file_kinds.rs` an_exfat_trash_that_stays_open_refuses_the_move (linux-mounts PASS); PR #117
- [x] The safety ADR §5 and §6 state the rules for special files, immutable flags, removal through descriptors, case, and exFAT/FAT, critic-approved. — evidence: `adr/cleanup-safety-model-risk-classes-plan-and-confirmation.md`, amendment of 2026-11-04 (vault PRs #98 and #102), critic-approved after three passes

## Final Summary

**What changed.** Four code PRs, all on main:
- #117 (released in 0.33.1): markers open only as regular files (`O_NOFOLLOW | O_NONBLOCK | O_NOCTTY`, then `fstat`); tombstone names cut on a character boundary with an `ENAMETOOLONG` retry; the suggested clean command keeps a newline or tab literal for dash, or suggests none; the Linux Trash re-checks its modes after `fchmod` and refuses an exFAT/FAT Trash that stays open, in the dry run too; device, FIFO, socket, sparse and odd-name tests and CI fixtures.
- #120 (0.34.1, tagged, not yet published): `.ddc-keep` in any ASCII case in the rule visitor, `keep_markers`, `walk_item` and the checks above an item; protected folders by fold and `(dev, ino)`; `keep remove` names a marker in another case instead of saying none is there.
- #124: immutable and append-only flags (`STATX_ATTR_IMMUTABLE`/`APPEND` on Linux, `UF_`/`SF_` flags on macOS) keep a finding, a picked entry and a worktree out of every plan, and the clean refuses them before any rename (`immutable`). Schemas: scan 9, clean 12, journal 12, progress 10.
- #125: a tombstone is removed through descriptors (`DirFd::remove_tree`), never by path: each level opens beneath its parent and must keep its device and mount id; the opened tombstone must be the `(dev, ino)` the guard checked; a stop leaves the tombstone and reports `failed`.

**Why.** A developer's home holds more than regular files: these kinds could hang a scan on a FIFO, read a device, make a folder that never empties from the Trash, or let a removal follow a mount made inside a finding.

**Tests executed.** Full native Linux leg on a Linux machine after each PR (last: 1086 passed, 161 skips as listed); CI Gate on every PR, linux-mounts included and green before each merge; DDCKit 493 passed for #124.

**Risks and follow-ups.**
- Releases now go in batches (the owner, 2026-11-04): #124 and #125 carry no bump. The 0.34.1 tag is unpublished because the notary keychain item is missing on the owner's Mac; the next release covers 0.34.1, #122, #123, #124 and #125.
- A folder made unreadable (`0000`) inside a finding after the scan now stops the removal and keeps the tombstone, where the old path-based chmod opened it. Fail-closed and rare; a line in the ADR's "What stops the descent" would make it explicit.
- The `openat` fallback of `remove_tree` (no `openat2`, as under an old seccomp profile) has no test of its own; the walk's fallback has one.
- The dry run and the clean can name different reasons for one refused item (the clean's walk before it finds a flag earlier than the dry run's checks); both refuse.
- `Action::Delete { make_writable }` and `Fallback::DeleteWritable` now mean the same as plain delete; kept so the plan's shape does not move.
- Manual check for the owner: none beyond the automated ones.

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

*Last updated: 2026-11-05*

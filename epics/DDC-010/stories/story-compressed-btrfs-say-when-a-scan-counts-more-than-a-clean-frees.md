---
type: story
id: "story-compressed-btrfs-say-when-a-scan-counts-more-than-a-clean-frees"
epic: "DDC-010"
title: "Compressed btrfs: say when a scan counts more than a clean frees"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-03
updated: 2026-11-03
external_refs: {}
tags: [linux, scanner, btrfs, compression]
code_refs: [crates/ddc-core/src/platform/linux/statfs.rs, crates/ddc-core/src/platform/linux/bulk.rs, scripts/ci-mounts.sh]
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Compressed btrfs: say when a scan counts more than a clean frees

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

On a compressed btrfs, a finding's `bytes` is the size before compression. The CI mounts measured it in the safety ADR's amendment "btrfs and ZFS on Linux", bullet "Compression, measured (2026-11-03)": 16 MiB of repeated text counted as 16 MiB, and the clean freed 512 KiB. That is synthetic data; for a real `node_modules` or `target/` the ratio is not known, only that `bytes` is larger by a ratio ddc does not know. Fedora mounts the home with `compress=zstd:1` by default, so this is the common case on the main btrfs desktop. The measurement and the schema's description of `bytes` come with DeepDriveCleaner PR #114 (story "btrfs and ZFS on Linux: subvolumes, datasets, snapshots and stable identities"); the report and the app say nothing.

ddc cannot get the on-disk size without root (`BTRFS_IOC_TREE_SEARCH`). It can tell, cheaply, that a size may be over-counted:
- the mount: mountinfo's `compress=` or `compress-force=` super option, already parsed into `Mount.super_options` in `platform/linux/statfs.rs`;
- the file or folder: `STATX_ATTR_COMPRESSED`, which btrfs sets from the inode's compress flag (`chattr +c`, `btrfs property set … compression`), and which the walk already reads through `stx_attributes` (`platform/linux/bulk.rs`). No extra open or ioctl.

ZFS needs nothing: its `st_blocks` is already the size on disk.

## Decomposition

- [ ] Decide the shape: a per-scan note (the root's mount compresses), a per-finding flag (a compressed inode inside it), or both. Amend the safety ADR's compression bullet and run the critic on it.
- [ ] Read the signals: the mount's super options, and `STATX_ATTR_COMPRESSED` from the statx the walk already does. `FS_IOC_GETFLAGS` only as a fallback where `stx_attributes_mask` lacks the bit; it needs a real open, not an O_PATH one.
- [ ] Report it in the scan JSON (a schema version move if it is a new field or enum value), in the CLI's table and in DDCKit, and say "may free less" next to the selected total.
- [ ] CI mounts: extend the `loop` helper in `scripts/ci-mounts.sh` with mount options; a new loop btrfs mounted `-o compress=zstd:1` carries the scan-level note; PR #114's `btrfs-zstd/c` (the property on a plain mount) carries the folder-level one; the plain btrfs and ZFS carry neither.

## Implementation Plan

## Acceptance Criteria

- [ ] A scan of a root on a btrfs mounted with `compress=` or `compress-force=` says that sizes may be over-counted; a scan of an uncompressed btrfs and of ZFS does not (CI mounts test, new `-o compress=zstd:1` loop).
- [ ] A folder with the compress inode flag on an uncompressed btrfs mount carries the note (CI mounts test, `btrfs-zstd/c`).
- [ ] The safety ADR's compression bullet carries the decision, critic-approved.

## Final Summary

## Technical Notes

- `compsize` gives the real numbers but needs root; ddc never runs sudo.
- A note, never a corrected number: ddc does not know the ratio. And "may": `compress=` applies only to data written after the mount, and `nodatacow` (`chattr +C`) files are never compressed.
- Out of scope, other reasons a clean frees less than `bytes`: blocks a snapshot still holds (Snapper, zsys; the schema's `bytes` description says so), and the Trash on the same filesystem, which frees nothing until it is emptied. The note must not be worded so that it seems to cover them.

## Dependencies

- Story "btrfs and ZFS on Linux: subvolumes, datasets, snapshots and stable identities" (DDC-010), which measured it; its PR #114 brings the `btrfs-zstd` fixture.

## Attachments

-

---

*Last updated: 2026-11-03*

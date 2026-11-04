---
type: story
id: "story-btrfs-and-zfs-on-linux-subvolumes-datasets-snapshots-and-stable-identities"
epic: "DDC-010"
title: "btrfs and ZFS on Linux: subvolumes, datasets, snapshots and stable identities"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-04
external_refs: {}
tags: [linux, scanner, safety, filesystems, btrfs, zfs]
code_refs: [crates/ddc-core/src/platform/linux/btrfs.rs, crates/ddc-core/src/platform/linux/fsid.rs, crates/ddc-core/src/platform/linux/bulk.rs, crates/ddc-core/src/platform/linux/fsops.rs, crates/ddc-core/src/platform/linux/meta.rs, crates/ddc-core/src/platform/linux/procs.rs, crates/ddc-core/src/platform/linux/statfs.rs, crates/ddc-core/src/platform/linux/trash.rs, crates/ddc-core/src/platform/linux/mod.rs, crates/ddc-core/src/platform/mod.rs, crates/ddc-core/src/platform/types.rs, crates/ddc-core/src/scan/walker.rs, crates/ddc-core/src/scan/mod.rs, crates/ddc-core/src/exec/guard.rs, crates/ddc-core/src/agents/liveness.rs, crates/ddc-core/src/model.rs, app/Packages/DDCKit/Sources/DDCKit/ScanReport.swift, docs/scan.schema.json, crates/ddc-core/tests/subvolumes.rs, crates/ddc-core/tests/datasets.rs, crates/ddc-core/tests/common/mounts.rs, scripts/ci-mounts.sh, .config/native-skips/linux.txt]
specs: []
started_at: null
closed_at: "2026-11-04T09:36:30.292Z"
plan_updated_at: null
---

# btrfs and ZFS on Linux: subvolumes, datasets, snapshots and stable identities

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | done |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

Fedora puts the home on btrfs by default. Ubuntu offers ZFS on root, and Proxmox uses it. Both filesystems give parts of one filesystem their own device numbers, and both keep snapshots that look like ordinary folders.

- **btrfs subvolumes.** Each subvolume has its own anonymous `st_dev` on the same filesystem. Today the walk stops at every subvolume as if it were another filesystem, as `du -x` does. So a subvolume inside the home is not counted. Examples are a Snapper `.snapshots` folder, a user-made `btrfs subvolume create`, and rootless Podman storage on btrfs.
- **btrfs snapshots** are subvolumes too. Entering them counts each snapshot as a full copy, and a `(dev, ino)` dedupe cannot catch that, because each snapshot has its own `dev`. A snapshot is not always read-only: Timeshift and a plain `btrfs subvolume snapshot` make writable ones. The unprivileged ioctl `BTRFS_IOC_GET_SUBVOL_INFO` (kernel 4.18 and later) returns `parent_uuid`, which is non-zero for a snapshot, and `flags`, with the read-only bit.
- **ZFS datasets** have their own `st_dev`, like subvolumes. Snapshots are in a hidden `.zfs/snapshot` folder at the dataset's root, reachable by path even when `snapdir=hidden`. `st_blocks` reflects compression. FIEMAP is not supported.
- **Stable identities.** `ddc` saves `(dev, ino)` pairs: in `last-scan.json`, in kept-folder records (`keep remove --folder-dev --folder-ino`) and in the plan's re-check. btrfs's anonymous device numbers depend on mount order and are not stable across mounts or boots. A record keyed on one can stop matching without a message, and a kept folder then loses its protection.

## Decomposition

- [x] Decide the subvolume and dataset rule, amend the safety ADR §5 and ADR "Platform seam" §9, and run the critic on the amendment. The proposal:
  - enter a btrfs subvolume of the same filesystem whose `parent_uuid` is zero, and a ZFS child dataset of the same pool;
  - never enter a snapshot: a btrfs subvolume with a non-zero `parent_uuid` (read-only or writable), or `.zfs/snapshot`. Report it as skipped with a new reason, `snapshot`, and never count or plan it;
  - keep `other-filesystem` for real other filesystems.
- [x] Detection, in this order for each folder.
  - **btrfs, mount check first.** Is the folder on the same mount as its parent? That is the same `stx_mnt_id` on 5.8 and later, or the same `/proc/self/mountinfo` entry. A different mount is another filesystem, even a second btrfs disk mounted at `~/data`, and the mount-boundary rule refuses it before the subvolume rule runs. `statfs.f_fsid` cannot tell "same filesystem" on btrfs: the kernel mixes the subvolume's root id into it.
  - **btrfs, then the subvolume.** Same mount, different `stx_dev` and btrfs: it is a subvolume reached by path. `openat2(RESOLVE_NO_XDEV)` lets it through, because it is no mount crossing. `GET_SUBVOL_INFO` then says whether it is a snapshot.
  - **ZFS.** Datasets are mounts, so the dataset rule runs before `RESOLVE_NO_XDEV`. A mount is allowed only when mountinfo's source is a child dataset of the scanned root's own dataset: its name starts with `<root dataset>/`, in the same pool. `.zfs` is recognised by name at a dataset root.
  - A btrfs folder where `GET_SUBVOL_INFO` is refused is reported as skipped with the existing reason `other-filesystem`, and is not counted. It is never reported as zero.
- [x] Persisted identities.
  - Audit every place that saves or compares `(dev, ino)`: `last-scan.json`, `keep`, the plan and the guard.
  - On btrfs, replace the anonymous `dev` with a stable key. Candidates: `(f_fsid, ino)`, since btrfs derives `f_fsid` from the on-disk fsid and the subvolume id, so it needs no ioctl; or the subvolume id (`GET_SUBVOL_INFO.treeid`).
  - Otherwise, re-verify by path, and say so when a record no longer matches.
  - Test across a remount of the loopback btrfs.
- [x] Compression. On btrfs with compression and on ZFS, `st_blocks` may differ from what a delete frees. Measure it on the CI mounts, state the result in the scan schema's description of `bytes`, and decide whether a note is needed.
- [x] The new reason `snapshot` and its schema version move belong to this story.
- [x] Tests on the CI mounts:
  - a subvolume is counted;
  - a read-only snapshot and a writable snapshot are skipped as `snapshot`;
  - a ZFS child dataset is counted, and `.zfs/snapshot` is skipped;
  - a kept folder on a subvolume stays kept after a remount.
  
  ZFS goes to the owner checks if the spike shows it cannot run on the hosted runner.

## Implementation Plan

Planned 2026-11-02 with the planner, from origin/main 018060d (0.26.0). The contract is the safety ADR's amendment "btrfs and ZFS on Linux", accepted 2026-11-02 after two critic passes. Three code PRs, each with its own bump:
1. **btrfs.**
   - `platform/linux/btrfs.rs` with `GET_SUBVOL_INFO` (a hand-written `#[repr(C)]` struct with a size assert; `unsafe` only under `platform/linux/`).
   - In `bulk.rs`'s `entry_meta`/`Mounts`, a subvolume is entered or skipped as `snapshot`. The walker's device check compares with the parent folder, not the root.
   - `holds-subvolume` makes the enclosing finding incomplete. `SkipReason::Snapshot` and `HoldsSubvolume` raise the scan schema, with DDCKit and the contract tests.
   - CI mounts: a plain and a nested subvolume, read-only, writable and received snapshots, a second btrfs inside.
2. **ZFS.**
   - `statfs.rs`'s `Mount` gains the parent id, root, source and mount point.
   - The five-condition child-dataset rule, hop by hop, with its own anchors in the walk (`Task` carries its anchor) and in the guard.
   - `.zfs` is skipped by name.
   - CI: a child and a grandchild, a bind of a dataset's mount point, and `snapdir=visible` with mountinfo unchanged.
3. **Stable identity.**
   - Every Linux device producer folds `f_fsid` on btrfs and ZFS: the walk's stat, `stat_child`, `identity`, `meta::lstat`/`stat`/`entry`, `procs::open_vnodes`, and the Trash's device and landed checks. `meta::file_id(&Metadata)` goes.
   - Tests: a remount on its own loop btrfs keeps a saved scan and a keep record matching, and a file held open inside a subvolume is in use.

The story's premise is corrected: a kept folder does not lose its protection after a remount (the marker is checked by path). The harm is that every saved report goes stale.


## Acceptance Criteria

- [x] On a loopback btrfs, a subvolume inside the root is counted per the decision. Read-only and writable snapshots are never counted or planned, and the scan reports them as `snapshot` (tests in the CI mounts job). — evidence: `crates/ddc-core/tests/subvolumes.rs` `a_plain_subvolume_is_entered_and_a_snapshot_never` (read-only, writable and received snapshots skipped as `snapshot`), job `linux-mounts` green on PRs #106, #112, #114 and on main's Gate for 1366228
- [x] On ZFS, a child dataset is handled per the decision, and `.zfs/snapshot` is never entered (CI test, or an owner check with the reason). — evidence: `crates/ddc-core/tests/datasets.rs` `a_zfs_child_dataset_is_entered_and_a_bind_of_it_never` and `a_visible_snapshot_folder_is_never_entered_and_mounts_nothing`, job `linux-mounts` on PR #110; ZFS runs on the hosted runner, so no owner-check fallback was needed
- [x] A kept folder and a saved scan on a btrfs subvolume still match after the filesystem is remounted (test). — evidence: `crates/ddc-core/tests/subvolumes.rs` `a_saved_scan_and_a_keep_record_still_match_after_a_remount`, fixture `btrfs-remount` in `scripts/ci-mounts.sh`, job `linux-mounts` on PR #112 and main
- [x] The safety ADR §5 and ADR "Platform seam" §9 carry the subvolume, dataset and snapshot rules, critic-approved. — evidence: safety ADR amendment "btrfs and ZFS on Linux" (2026-11-02, two critic passes) with its correction and the bullet "Compression, measured (2026-11-03)" (critic-approved, vault PR #92). Correction to this criterion: "Platform seam" §9 is about tests on a platform that is not supported yet, so the reference was wrong; nothing platform-seam changed (no new capability, no new seam function), and the rules live only in the safety ADR.

## Final Summary

**What changed.** Four code PRs, each released:
- #106 (0.28.0, musl fix in 0.28.1): btrfs. `platform/linux/btrfs.rs` reads `BTRFS_IOC_GET_SUBVOL_INFO`. A same-mount subvolume is entered; a snapshot (non-zero `parent_uuid`) is skipped as `snapshot`. A finding that holds a subvolume is `holds-subvolume` and not planned.
- #110 (0.29.0): ZFS. A child dataset is entered only under the five conditions of the ADR, each with its own anchor in the walk and the guard. `.zfs` is skipped by name.
- #112 (0.30.0): stable identity. Every Linux producer of a device folds `f_fsid` on btrfs and ZFS through `platform/linux/fsid.rs`, cached per (mount id, raw dev). `/proc/pid/fd` links are opened once with O_PATH. The scan schema moved to 8.
- #114 (0.30.2): compression. Measured on the CI mounts (btrfs zstd: 16 MiB of text counted, 512 KiB freed; ZFS lz4: size on disk). Written in the `bytes` descriptions of the schema and in the ADR.

**Why.** Fedora's home is btrfs and Ubuntu and Proxmox offer ZFS. Before this, ddc stopped at every subvolume and dataset, would have counted snapshots as copies, and lost every saved scan at a remount.

**Tests executed.**
- CI job `linux-mounts` on real loop btrfs and ZFS on PRs #106, #110, #112 and #114, and on main's Gate.
- Native Linux runs on the Windows host's WSL: 977 of 977 tests, 151 listed skips.
- Reviewer passes on PR 3 and PR 4, critic passes on the ADR amendments.

**Risks and follow-ups.**
- Owner checks: "btrfs and ZFS on Linux (DDC-010)" in the runbook "Owner checks: manual acceptance across stories" (real Snapper home, reboot identity, compression on Fedora, an Ubuntu ZFS home).
- Story "Compressed btrfs: say when a scan counts more than a clean frees" (DDC-010, p2): the report does not flag compression yet.
- `execute` does not re-walk a finding before removal.
- The recent-lock rule reads a file's birth time while `child_times` reads mtime; they can disagree.
- `linux-schedule` can become a required check after 2026-11-09 if it stays green.
- A subvolume deleted and re-created inside one process can reuse its anonymous device, and the fold cache then returns the old fsid. The result is a stale scan, never an unsafe one.

## Technical Notes

- Rootless Docker uses overlay2, fuse-overlayfs or vfs, not the btrfs driver. Rootless Podman can use btrfs. Root Docker's btrfs storage is in `/var/lib/docker`, which is never walked.
- `du -x` stops at subvolumes. The comparison tests need a note where the decision differs.

## Dependencies

- Story "Linux CI mounts: a filesystem matrix job with root-only steps".
- Story "Linux mount boundaries: automounts, bind mounts and the Trash across filesystems". Its mount check runs before the btrfs subvolume rule. `RESOLVE_NO_XDEV` refuses a ZFS dataset mount, so the ZFS dataset rule runs before it.

## Attachments

-

---

*Last updated: 2026-11-04*

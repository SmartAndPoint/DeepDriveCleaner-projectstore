---
type: story
id: "story-btrfs-and-zfs-on-linux-subvolumes-datasets-snapshots-and-stable-identities"
epic: "DDC-010"
title: "btrfs and ZFS on Linux: subvolumes, datasets, snapshots and stable identities"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [linux, scanner, safety, filesystems, btrfs, zfs]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# btrfs and ZFS on Linux: subvolumes, datasets, snapshots and stable identities

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | planned |
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

- [ ] Decide the subvolume and dataset rule, amend the safety ADR §5 and ADR "Platform seam" §9, and run the critic on the amendment. The proposal:
  - enter a btrfs subvolume of the same filesystem whose `parent_uuid` is zero, and a ZFS child dataset of the same pool;
  - never enter a snapshot: a btrfs subvolume with a non-zero `parent_uuid` (read-only or writable), or `.zfs/snapshot`. Report it as skipped with a new reason, `snapshot`, and never count or plan it;
  - keep `other-filesystem` for real other filesystems.
- [ ] Detection, in this order for each folder.
  - **btrfs, mount check first.** Is the folder on the same mount as its parent? That is the same `stx_mnt_id` on 5.8 and later, or the same `/proc/self/mountinfo` entry. A different mount is another filesystem, even a second btrfs disk mounted at `~/data`, and the mount-boundary rule refuses it before the subvolume rule runs. `statfs.f_fsid` cannot tell "same filesystem" on btrfs: the kernel mixes the subvolume's root id into it.
  - **btrfs, then the subvolume.** Same mount, different `stx_dev` and btrfs: it is a subvolume reached by path. `openat2(RESOLVE_NO_XDEV)` lets it through, because it is no mount crossing. `GET_SUBVOL_INFO` then says whether it is a snapshot.
  - **ZFS.** Datasets are mounts, so the dataset rule runs before `RESOLVE_NO_XDEV`. A mount is allowed only when mountinfo's source is a child dataset of the scanned root's own dataset: its name starts with `<root dataset>/`, in the same pool. `.zfs` is recognised by name at a dataset root.
  - A btrfs folder where `GET_SUBVOL_INFO` is refused is reported as skipped with the existing reason `other-filesystem`, and is not counted. It is never reported as zero.
- [ ] Persisted identities.
  - Audit every place that saves or compares `(dev, ino)`: `last-scan.json`, `keep`, the plan and the guard.
  - On btrfs, replace the anonymous `dev` with a stable key. Candidates: `(f_fsid, ino)`, since btrfs derives `f_fsid` from the on-disk fsid and the subvolume id, so it needs no ioctl; or the subvolume id (`GET_SUBVOL_INFO.treeid`).
  - Otherwise, re-verify by path, and say so when a record no longer matches.
  - Test across a remount of the loopback btrfs.
- [ ] Compression. On btrfs with compression and on ZFS, `st_blocks` may differ from what a delete frees. Measure it on the CI mounts, state the result in the scan schema's description of `bytes`, and decide whether a note is needed.
- [ ] The new reason `snapshot` and its schema version move belong to this story.
- [ ] Tests on the CI mounts:
  - a subvolume is counted;
  - a read-only snapshot and a writable snapshot are skipped as `snapshot`;
  - a ZFS child dataset is counted, and `.zfs/snapshot` is skipped;
  - a kept folder on a subvolume stays kept after a remount.
  
  ZFS goes to the owner checks if the spike shows it cannot run on the hosted runner.

## Implementation Plan

## Acceptance Criteria

- [ ] On a loopback btrfs, a subvolume inside the root is counted per the decision. Read-only and writable snapshots are never counted or planned, and the scan reports them as `snapshot` (tests in the CI mounts job).
- [ ] On ZFS, a child dataset is handled per the decision, and `.zfs/snapshot` is never entered (CI test, or an owner check with the reason).
- [ ] A kept folder and a saved scan on a btrfs subvolume still match after the filesystem is remounted (test).
- [ ] The safety ADR §5 and ADR "Platform seam" §9 carry the subvolume, dataset and snapshot rules, critic-approved.

## Final Summary

## Technical Notes

- Rootless Docker uses overlay2, fuse-overlayfs or vfs, not the btrfs driver. Rootless Podman can use btrfs. Root Docker's btrfs storage is in `/var/lib/docker`, which is never walked.
- `du -x` stops at subvolumes. The comparison tests need a note where the decision differs.

## Dependencies

- Story "Linux CI mounts: a filesystem matrix job with root-only steps".
- Story "Linux mount boundaries: automounts, bind mounts and the Trash across filesystems". Its mount check runs before the btrfs subvolume rule. `RESOLVE_NO_XDEV` refuses a ZFS dataset mount, so the ZFS dataset rule runs before it.

## Attachments

-

---

*Last updated: 2026-11-01*

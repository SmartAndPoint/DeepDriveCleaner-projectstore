---
type: story
id: "story-linux-ci-mounts-a-filesystem-matrix-job-with-root-only-steps"
epic: "DDC-010"
title: "Linux CI mounts: a filesystem matrix job with root-only steps"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [linux, tests, ci, filesystems]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Linux CI mounts: a filesystem matrix job with root-only steps

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

Every Linux test runs on one filesystem today: the CI runner's ext4, or the overlay of the native-runs container. Both run as a user without root. Several stories of this epic can only prove their criteria on other filesystems, or with a root-only setup step:
- the mount boundaries;
- btrfs and ZFS;
- network homes;
- file kinds (`chattr +i`, `mknod`);
- private size (reflinks).

Under the owner's rule, a story is done only when its automated criteria have evidence. So this job comes first. It is infrastructure only: no product code changes here.

The job runs on GitHub's Ubuntu runner itself, which has passwordless `sudo`, and not in a container. A container would need `--privileged` to mount.

## Decomposition

- [ ] Spike first, in a throwaway workflow on a branch, and record the answers in this story:
  - loop devices and `mkfs.btrfs`, `mkfs.xfs` on `ubuntu-24.04`;
  - `zfsutils-linux` and a file-backed pool;
  - `nfs-kernel-server` with a loopback export, including `all_squash`;
  - `mknod` and `chattr +i` under `sudo`;
  - an ext4 image mounted with `-O casefold` and a folder with `chattr +F`;
  - an exFAT image.
  
  Each one that does not work on the hosted runner becomes an owner check, with the reason.
- [ ] `scripts/ci-mounts.sh`. It makes the mounts the spike confirmed under one folder, owned by the runner user, and exports each one's path:
  - btrfs with a subvolume, a read-only snapshot and a writable snapshot;
  - XFS;
  - tmpfs, twice: one owned by the runner user, and one whose top is owned by root, with a user-owned folder inside it for the cross-filesystem Trash refusal;
  - ZFS with a child dataset and a snapshot;
  - loopback NFS, plain and `all_squash`;
  - casefold ext4;
  - exFAT;
  - a bind mount of the runner's ext4 inside a test root.
  
  The exported variables are `DDC_MOUNT_BTRFS`, `DDC_MOUNT_XFS` and so on. It also prepares the root-only fixtures that tests read by path from variables: a FIFO, a device node, an immutable file and an append-only file.
- [ ] A test helper `common::mount(kind)`. It returns the path from the variable, or skips with the reason `no <kind> mount`. Native runs and macOS skip it.
- [ ] `.config/native-skips/linux.txt` gains a group "environment: needs the CI mounts". `scripts/native-skips.sh` accepts a name of that group as skipped in native runs and refuses it as skipped in the mounts job.
- [ ] The job `linux-mounts` in `gate.yml`:
  - runs the tests that use `common::mount`, through a nextest filter;
  - runs on main, nightly, and on a branch when `crates/ddc-core/src/platform/linux/`, `scan/`, `exec/` or the script changed (`scripts/gate-scope.sh`).
  
  Not a required check until it has been green for a week.
- [ ] The native-runs runbook: an optional local step for a Linux machine where loop mounts are allowed.

## Implementation Plan

## Acceptance Criteria

- [ ] The spike's answers are recorded here: which mount types work on the hosted runner, and which became owner checks.
- [ ] The `linux-mounts` job is green on main and names each mount in its log. One test per mount type proves that the mount is really that filesystem (`statfs` `f_type`).
- [ ] A test that uses `common::mount` skips with its reason in native runs, and `scripts/native-skips.sh` accepts this only for the environment group (test of the script).

## Final Summary

## Technical Notes

- `chattr -i` is root-only too. The prepared immutable file stays on the runner, which is discarded after the job.
- Keep the mounts small (a few hundred MB). The job must finish in minutes.

## Dependencies

- None. The mount, btrfs and ZFS, network-home, file-kind and private-size stories depend on this one.

## Attachments

-

---

*Last updated: 2026-11-01*

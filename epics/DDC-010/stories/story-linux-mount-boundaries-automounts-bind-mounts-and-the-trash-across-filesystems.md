---
type: story
id: "story-linux-mount-boundaries-automounts-bind-mounts-and-the-trash-across-filesystems"
epic: "DDC-010"
title: "Linux mount boundaries: automounts, bind mounts and the Trash across filesystems"
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

# Linux mount boundaries: automounts, bind mounts and the Trash across filesystems

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

On macOS the walk stays on one volume. APFS gives each volume its own device number, so `dev` alone marks the boundary. On Linux the walk compares `stx_dev` only (`platform/linux/bulk.rs`), and three cases break that rule.

- **Bind mounts.** A bind mount of the same filesystem has the same `stx_dev`. A developer bind-mounts a folder from elsewhere into a project, such as a shared data folder or a container's volume. Today the walk enters it, counts it, and a rule can match inside it. A clean then deletes through the bind mount, into a folder outside the scanned root. On macOS this cannot happen.
- **Automounts.** An automount point is mounted by a `stat` or an `open`. The walk calls `statx` without `AT_NO_AUTOMOUNT`, and `openat` on a directory triggers the mount. With an autofs map under the home, which is common on university and corporate machines, a scan mounts every share in the map. It can also hang on an unreachable server.
- **The Trash across filesystems.** The `trash` crate (5.2.9, `freedesktop.rs`) first tries a rename. When the rename crosses filesystems and no `.Trash-$uid` can be made at the volume's top, the crate copies every regular file into the home Trash with `std::fs::copy`, then removes the source. The safety ADR's §5 amendment of 2026-10-31 accepted this as the platform Trash's behaviour. This epic makes it common: caches moved to a second disk (story "Cache roots moved by the environment: XDG and each tool's own variable"), bind mounts and network homes. The copy causes four problems:
  - it fills the home disk with a second disk's finding;
  - it turns a hard-linked tree (uv, pnpm) into full copies;
  - it reads FIFOs and device nodes;
  - when it runs out of space halfway, it leaves a partial copy.

## Decomposition

- [ ] Directory opens with `openat2` and `RESOLVE_NO_XDEV | RESOLVE_NO_SYMLINKS | RESOLVE_BENEATH` (kernel 5.6 and later; `rustix` has it), in the walk, the verified-parent operations and the guard's re-walk. The kernel then refuses to cross any mount, bind mounts included, with `EXDEV`. That folder is reported as skipped with the existing reason `other-filesystem`. Where `openat2` is missing (`ENOSYS`), or refused with `EPERM` by an old seccomp profile in a container, fall back to the `statx` checks below, as `bulk.rs` already does for `statx`.
- [ ] Before any open, `statx` with `AT_NO_AUTOMOUNT`, reading `stx_attributes`:
  - a folder with `STATX_ATTR_AUTOMOUNT` is never opened, and is reported as skipped with a new reason, `automount`;
  - a folder with `STATX_ATTR_MOUNT_ROOT` (kernel 5.8 and later) is a boundary.
  
  On kernels older than 5.8, the fallback is the mount points in `/proc/self/mountinfo`, read once per scan (`platform/linux/statfs.rs` parses it already). `stx_mnt_id` arrived in 5.8 too, so it is no fallback.
- [ ] Check in the kernel source whether `openat2` with `RESOLVE_NO_XDEV` stops an autofs trigger or only refuses after the mount, and record the answer. The `STATX_ATTR_AUTOMOUNT` pre-check stays either way.
- [ ] The Trash across filesystems.
  - Before the move, compare the finding's filesystem with the home's.
  - When they differ, `ddc` does the freedesktop move itself, without the `trash` crate:
    - create `$topdir/.Trash-$uid` with `files/` and `info/`, mode 0700, through the verified parent;
    - write the `.trashinfo`;
    - `renameat` into `files/<unique>`.
    
    If any step fails, refuse the move with a new reason, `trash-crosses-filesystem`, and name `--permanent` in the message.
  - The crate stays for the home filesystem only, where its move is a rename. It picks its trash folder from its own mount-table view, so it cannot be relied on to avoid its copy fallback across filesystems.
  - After every Trash move, verify by `(dev, ino)` that the item is still on its source filesystem. Anything else is a bug that the tests catch.
  - Amend the safety ADR §5, replacing the acceptance of 2026-10-31, and run the critic on the amendment.
- [ ] The new reasons `automount` and `trash-crosses-filesystem`, and their schema version moves, belong to this story. The docs schemas and DDCKit follow.
- [ ] Tests, with the mounts from story "Linux CI mounts: a filesystem matrix job with root-only steps":
  - a bind mount inside a scanned root is skipped, never counted and never planned;
  - a guard re-walk of a stale plan through a bind mount refuses with `EXDEV`;
  - a Trash move from the root-owned tmpfs (no writable top) refuses with `trash-crosses-filesystem`;
  - a Trash move from a user-owned tmpfs uses `.Trash-$uid` there, with a valid `.trashinfo`;
  - after either, nothing of the item exists on the home filesystem.
  
  A unit test covers the automount pre-check with a fake `statx`. A real autofs map goes to the owner-checks runbook.

## Implementation Plan

## Acceptance Criteria

- [ ] A bind mount of the same filesystem inside a scanned root is reported as skipped, is never counted, and nothing inside it is ever planned or deleted (test on a real bind mount in the CI mounts job).
- [ ] A folder that carries `STATX_ATTR_AUTOMOUNT` is never opened, and the scan reports it as `automount` (unit test with a fake `statx`; an owner check with a real autofs map).
- [ ] No clean copies a file into the Trash. A move across filesystems uses that filesystem's `.Trash-$uid` or refuses with `trash-crosses-filesystem` (tests in the CI mounts job).
- [ ] The safety ADR §5 states the Linux mount rules and the cross-filesystem Trash rule, critic-approved.

## Final Summary

## Technical Notes

- `du -x` enters bind mounts of the same filesystem. Here the walk differs from `du` on purpose. The `du -skx` comparison tests need a note where a bind mount is present.
- WSL's `/mnt/c` is a 9p or drvfs mount. The walk from the home never enters it.

## Dependencies

- Story "Linux CI mounts: a filesystem matrix job with root-only steps".

## Attachments

-

---

*Last updated: 2026-11-01*

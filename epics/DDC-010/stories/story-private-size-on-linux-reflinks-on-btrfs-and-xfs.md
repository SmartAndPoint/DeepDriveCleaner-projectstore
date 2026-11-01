---
type: story
id: "story-private-size-on-linux-reflinks-on-btrfs-and-xfs"
epic: "DDC-010"
title: "Private size on Linux: reflinks on btrfs and XFS"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [linux, scanner, sizes, capability]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Private size on Linux: reflinks on btrfs and XFS

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

**Hard links are counted already, on every OS.**
- The walk records each `(dev, ino)` with its `nlink` and its holders (`scan/inodes.rs`).
- A finding's `bytes` holds only inodes that it alone holds and whose links were all seen. The rest is `shared_bytes` (`scan/walker.rs`, since 2026-10-17).
- The text report prints "N shared with other folders".

So a uv venv hard-linked to the uv cache already reports little in `bytes` on ext4. The research note's Conclusion 5 ("flag findings whose files have links outside them") asked for something that existed. It measured with separate `du` calls, and this story corrects the note.

**What Linux lacks is the clone half.** On macOS a finding where clones are expected (uv venvs, pnpm's `node_modules`) carries `private_bytes`. This is APFS's private size: a lower bound of what deleting the finding frees, read in the same bulk call as the walk (`ATTR_CMNEXT_PRIVATESIZE`). The plan's estimate uses it. On Linux the `private-size` capability is absent, so `private_bytes` is never present.

On ext4 nothing is missing, because ext4 cannot reflink. On btrfs and XFS, current uv and pnpm make reflinks. Their blocks are shared like APFS clones, and `bytes` counts them in full.

On btrfs and XFS, `FS_IOC_FIEMAP` marks shared extents with `FIEMAP_EXTENT_SHARED`. The private size of a file is the sum of its extents not marked shared. Unlike the walk, FIEMAP needs an open file descriptor for each file. Today the Linux walk never opens a regular file.

## Decomposition

- [ ] An erratum in the research note's Conclusion 5 and §7: hard links were already counted, and the open part is reflinks.
- [ ] FIEMAP sharing.
  - Open each regular file of a finding where clones are expected (the rules' clone markers: uv venvs, pnpm) with `O_RDONLY | O_NOFOLLOW | O_NOATIME`, through the verified parent. `O_NOATIME` fails with `EPERM` on a file the user does not own. Retry without it there. Read its extent map and close it.
  - Do this only on btrfs and XFS (by `f_type`), and only when the caller asks for private size (`want_private`), as on macOS.
  - Extents marked `FIEMAP_EXTENT_SHARED` are not private.
- [ ] The opens are visible to other tools. The in-use check reads other processes only, so these opens do not count against the finding: test it. On a network filesystem, skip FIEMAP.
- [ ] Cost.
  - On btrfs, FIEMAP with shared-extent flags walks back-references and can be slow.
  - Cap it with an entry budget. Past the budget, leave `private_bytes` absent, as macOS does when the value is not returned.
  - Measure on the load trees with `make load` and the Linux limits.
- [ ] Decide what `private_bytes` means when a venv and the cache it shares with are both findings in one scan. Deleting both frees the sum; deleting one frees its private part. Write the decision into the scan schema's description.
- [ ] Declare `private-size` on Linux.
  - Update the "when absent" column of ADR "Platform seam: one platform module per OS with declared capabilities", §3.
  - Update the `private-size` and `private_bytes` descriptions in `docs/scan.schema.json`, which name APFS only.
  - The `private_size_is_unknown_when_not_returned` test and its kin leave the Linux skip list.
- [ ] Hints. The shared `cache.pnpm` hint, which says cloned files "stay intact", gets a Linux wording (research note §7).
- [ ] Tests on the CI mounts:
  - a pnpm store reflinked into a project on btrfs and on XFS: `private_bytes` is the unshared part;
  - on ext4 and tmpfs, `private_bytes` is absent;
  - the plan's estimate uses `private_bytes` where present.

## Implementation Plan

## Acceptance Criteria

- [ ] On btrfs and XFS, a reflinked copy of a cache has `private_bytes` equal to its unshared extents, and the plan's estimate uses it (tests in the CI mounts job).
- [ ] FIEMAP runs only for expected-clone findings on reflink filesystems, within a budget. Past the budget, `private_bytes` is absent (test).
- [ ] `CAPABILITIES` for Linux holds `private-size`. The scan schema and ADR "Platform seam" §3 describe it for both platforms.
- [ ] The Linux S load check stays within `bench/load-limits-linux.tsv` with private size on (nightly `load-linux`).
- [ ] The research note carries the erratum.

## Final Summary

## Technical Notes

- `FIEMAP_EXTENT_SHARED` is reported by btrfs, XFS and OCFS2. A snapshot also keeps shared blocks alive, which is right for a finding: deleting it does not free them.
- ZFS does not support FIEMAP, and its block cloning is not visible this way. `private_bytes` stays absent there.

## Dependencies

- Story "Linux CI mounts: a filesystem matrix job with root-only steps".
- Story "btrfs and ZFS on Linux: subvolumes, datasets, snapshots and stable identities", for the snapshot rule.

## Attachments

-

---

*Last updated: 2026-11-01*

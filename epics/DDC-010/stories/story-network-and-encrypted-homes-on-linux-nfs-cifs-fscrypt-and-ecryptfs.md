---
type: story
id: "story-network-and-encrypted-homes-on-linux-nfs-cifs-fscrypt-and-ecryptfs"
epic: "DDC-010"
title: "Network and encrypted homes on Linux: NFS, CIFS, fscrypt and eCryptfs"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [linux, scanner, safety, filesystems, nfs]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Network and encrypted homes on Linux: NFS, CIFS, fscrypt and eCryptfs

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

University and corporate machines often mount the home over NFS or CIFS. The research note [[research/where-developer-disk-space-goes-linux|Where a developer's disk space goes (Linux)]] lists this as an open question: uid mapping, `stx_dev` and latency were not measured.

- **Ownership.** Under `all_squash`, or when the client's and the server's uids differ, files show another owner, often `nobody`. `root_squash` maps only root's own accesses. Today's ownership check would refuse each such entry as `foreign-owner`, one line each.
- **Silly renames.** Deleting a file that a process still holds open makes the NFS client rename it to `.nfsXXXX`, which stays until the file is closed.
- **CIFS** has no Unix owners unless mapped, and its names may be case-insensitive.
- **Latency.** Each `statx` costs a round trip. `AT_STATX_DONT_SYNC` is already passed.
- **Encrypted homes.** fscrypt and the older eCryptfs (`~/.Private`, `/home/.ecryptfs`). The scan must read only the unlocked view.

## Decomposition

- [ ] Detect a network filesystem from `statfs`'s `f_type` (NFS, CIFS/SMB2). Every FUSE mount shares one `f_type`, so sshfs is told apart from gocryptfs and the like by mountinfo's filesystem type (`fuse.sshfs`).
- [ ] Ownership.
  - When most entries under a root have an owner the user is not, report it once as a note on the root ("files are owned by `<uid>`: the server maps ids"), and not as thousands of `foreign-owner` lines.
  - Nothing the user cannot delete is planned.
  - Say in the text report when `--permanent` may be the only working clean.
- [ ] Silly renames. `.nfs*` files are never counted as new findings after a clean. The clean report names a delete that left one.
- [ ] Measure a scan on the loopback NFS export of the CI mounts job, and record the latency in the research note's open question.
- [ ] Encrypted homes: the eCryptfs lower folder (`~/.Private`, `/home/.ecryptfs`) is never walked. Add `~/.Private` to the Linux protected folders through the safety ADR §6, unless it is already covered.
- [ ] Tests on the CI mounts job's NFS export, plain and `all_squash`:
  - a scan finishes and reports the mapping once;
  - a clean of a selected cache either removes it or refuses it with a reason, and leaves no tombstone.

## Implementation Plan

## Acceptance Criteria

- [ ] On a loopback NFS export with `all_squash`, a scan finishes and reports the id mapping once, not per entry (test in the CI mounts job).
- [ ] A clean on that export removes a selected cache or refuses it with a reason, and leaves no tombstone. A `.nfs*` file left by an open file is named in the report and not counted later (tests).
- [ ] The research note records the measured NFS latency, without describing any owner machine.
- [ ] `~/.Private` is never walked (test).

## Final Summary

## Technical Notes

- The Trash on a network home follows the cross-filesystem rule of story "Linux mount boundaries: automounts, bind mounts and the Trash across filesystems". The home Trash lives on the same NFS filesystem, so a move is a rename there.

## Dependencies

- Story "Linux CI mounts: a filesystem matrix job with root-only steps", for the NFS export.

## Attachments

-

---

*Last updated: 2026-11-01*

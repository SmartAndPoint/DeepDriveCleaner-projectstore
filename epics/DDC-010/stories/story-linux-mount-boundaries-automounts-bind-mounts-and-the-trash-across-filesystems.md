---
type: story
id: "story-linux-mount-boundaries-automounts-bind-mounts-and-the-trash-across-filesystems"
epic: "DDC-010"
title: "Linux mount boundaries: automounts, bind mounts and the Trash across filesystems"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [linux, scanner, safety, clean, filesystems]
code_refs: []
specs: []
started_at: "2026-11-01T18:37:21.133Z"
closed_at: null
plan_updated_at: "2026-11-01T18:37:21.133Z"
---

# Linux mount boundaries: automounts, bind mounts and the Trash across filesystems

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | in-progress |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

On macOS the walk stays on one volume. APFS gives each volume its own device number, so `dev` alone marks the boundary. On Linux the walk compares `stx_dev` only (`platform/linux/bulk.rs`), and three cases break that rule.

- **Bind mounts.** A bind mount of the same filesystem has the same `stx_dev`. A developer bind-mounts a folder from elsewhere into a project, such as a shared data folder or a container's volume. Today the walk enters it, counts it, and a rule can match inside it. A clean then deletes through the bind mount, into a folder outside the scanned root. On macOS this cannot happen.
- **Automounts.** An automount point is mounted by a `stat` or an `open`. The walk calls `statx` without `AT_NO_AUTOMOUNT`, and `openat` on a directory triggers the mount. With an autofs direct map under the home, which is common on university and corporate machines, a scan mounts every share in the map, and an NFS referral inside an entered mount is mounted too. It can also hang on an unreachable server. (An indirect map's root is another filesystem, so today's `dev` check already keeps the walk from listing its triggers.)
- **The Trash across filesystems.** The `trash` crate (5.2.9, `freedesktop.rs`) first tries a rename. When the rename crosses filesystems and no `.Trash-$uid` can be made at the volume's top, the crate copies every regular file into the home Trash with `std::fs::copy`, then removes the source. The safety ADR's §5 amendment of 2026-10-31 accepted this as the platform Trash's behaviour. This epic makes it common: caches moved to a second disk (story "Cache roots moved by the environment: XDG and each tool's own variable"), bind mounts and network homes. The copy causes four problems:
  - it fills the home disk with a second disk's finding;
  - it turns a hard-linked tree (uv, pnpm) into full copies;
  - it reads FIFOs and device nodes;
  - when it runs out of space halfway, it leaves a partial copy.

## Decomposition

- [ ] Directory opens with `openat2` and `RESOLVE_NO_XDEV | RESOLVE_NO_SYMLINKS | RESOLVE_BENEATH` (kernel 5.6 and later; `rustix` has it), in the walk, the verified-parent operations and the guard's re-walk. The kernel then refuses to cross any mount, bind mounts included, with `EXDEV`. That folder is reported as skipped with the existing reason `other-filesystem`. Where `openat2` is missing (`ENOSYS`), or refused with `EPERM` by an old seccomp profile in a container, fall back to the `statx` checks below, as `bulk.rs` already does for `statx`.
- [ ] Before any open, `statx` with `AT_NO_AUTOMOUNT`, reading `stx_attributes`:
  - a folder with `STATX_ATTR_AUTOMOUNT`, or a mount root whose mountinfo type is `autofs`, is never opened, and is reported as skipped with a new reason, `automount`. autofs marks its dentries, not the inode, so `STATX_ATTR_AUTOMOUNT` alone misses it;
  - a folder with `STATX_ATTR_MOUNT_ROOT` (kernel 5.8 and later) is a boundary.
  
  On kernels older than 5.8, the fallback is the mount points in `/proc/self/mountinfo`, read once per scan (`platform/linux/statfs.rs` parses it already). `stx_mnt_id` arrived in 5.8 too, so it is no fallback.
- [x] Check in the kernel source whether `openat2` with `RESOLVE_NO_XDEV` stops an autofs trigger or only refuses after the mount, and record the answer. The `STATX_ATTR_AUTOMOUNT` pre-check stays either way. — done at planning: on a kernel without commit 042a60680de4 (mainline 6.18, marked for the stable kernels) it triggers the mount first, so the `statx` pre-check is the guarantee on today's distribution kernels (Technical Notes)
- [ ] The Trash across filesystems, as the safety ADR amendment of 2026-11-01 lists it, steps 1 to 7.
  - Every Linux Trash move is `ddc`'s own freedesktop move; the `trash` crate is removed from the Linux target. Planning found that the crate also copies on the home's own mount: a rename across btrfs subvolumes, or on overlayfs, gives `EXDEV` there too.
  - No usable trash on the item's filesystem, or `EXDEV` from the rename, skips the item with a new reason, `trash-crosses-filesystem`, whose message names `--permanent`. Any other failure is `failed` with its error. The rename falls back to a plain `renameat` where the filesystem refuses `RENAME_NOREPLACE` (NFS).
  - The dry run runs the same selection with nothing created.
  - The journal records where each item went.
  - Amend the safety ADR §5, replacing the acceptance of 2026-10-31, and run the critic on the amendment. — amendment of 2026-11-01 written at the work-start gate, critic-approved after two passes
- [ ] The new reasons `automount` and `trash-crosses-filesystem`, and their schema version moves, belong to this story. The docs schemas and DDCKit follow.
- [ ] Tests, with the mounts from story "Linux CI mounts: a filesystem matrix job with root-only steps":
  - a bind mount inside a scanned root is skipped, never counted and never planned;
  - a guard re-walk of a stale plan through a bind mount refuses with `EXDEV`;
  - a Trash move from the root-owned tmpfs (no writable top) refuses with `trash-crosses-filesystem`;
  - a Trash move from a user-owned tmpfs uses `.Trash-$uid` there, with a valid `.trashinfo`;
  - after either, nothing of the item exists on the home filesystem;
  - an item in a btrfs subvolume other than the trash's refuses with `trash-crosses-filesystem`, where the crate would have copied it.
  
  - an autofs direct map in the CI mounts job: a scan of its parent leaves `/proc/self/mountinfo` without a new mount, and reports the trigger as `automount`;
  - an NFS export: a Trash move into its `.Trash-$uid` succeeds without the rename flag.
  
  A unit test covers `boundary()`. A real autofs map under a person's home goes to the owner-checks runbook.

## Implementation Plan

Planned 2026-11-01 with the planner, from origin/main 02ddbd8 (0.15.2). The safety ADR §5 amendment of 2026-11-01 is the contract. Two code PRs, each with its own version bump.

**PR A, the walk** (scan schema 5):
1. `platform/linux/errno.rs` maps `EXDEV` to `ReadErrorKind::OtherFilesystem`.
2. `platform/linux/statfs.rs` keeps each mountinfo line's mount id. One parse per scan gives the mount points and their types: the autofs check, and the fallback below kernel 5.8.
3. `platform/linux/bulk.rs`:
   - each entry's `statx` adds `AT_NO_AUTOMOUNT` and `STATX_MNT_ID`;
   - a pure `boundary(attrs, mask, entry_mnt, dir_mnt)` gives `Automount`, `Mount` or none;
   - `EntryMeta` gains `boundary: Option<Boundary>`, which is `None` on macOS, in `StdReader` and in the fakes.
4. Opens beneath the root.
   - `DirFd` exposes its descriptor.
   - New pinned calls: `bulk::list_in(root_fd, root, rel, …)` and `DirFd::open_beneath(root, rel)`. On Linux they use `openat2(RESOLVE_BENEATH | NO_XDEV | NO_SYMLINKS)` with a one-time probe and a path fallback. On macOS they keep today's open by path. `unported.rs` gets stubs.
   - `DirReader::read_in` defaults to `read`, so the 13 fake readers need no change. `SystemReader` overrides it.
   - `RootAcc` holds the root's descriptor.
5. The walker's child loop checks `Automount`, then `Mount`, then today's `dev`. The first two skip as `automount` and `other-filesystem` and count as `other_fs`. `guard.rs`'s walk takes `Automount` as `other_fs` too.
6. `SkipReason::Automount`, the scan schema 5, `docs/scan.schema.json`, DDCKit `ScanReport` and the goldens' version lines. Notes in the `du` docs on the deliberate difference from `du -x`.
7. `scripts/ci-mounts.sh` gains an autofs direct map (`autofs`, a `/-` map binding a folder of the base), exported as `DDC_MOUNT_AUTOFS`. `statfs(2)` and `stat -f` trigger an automount, so the script lists it from `/proc/self/mountinfo`, and `common::mount` checks this kind there and never with `statfs`. Exactly one test touches the trigger.
8. Tests:
   - unit tests of `boundary()`, of `EXDEV`, and of the `openat2` fallback with `EPERM` injected: the fallback descriptor must show the root's mount id and no `MOUNT_ROOT`;
   - on `Mount::Autofs`, a scan of the map's parent leaves `/proc/self/mountinfo` without a new mount, and the trigger is reported as `automount`;
   - a walker test with a fake reader marking `Automount` and `Mount`;
   - on `Mount::Bind`, a scan of the bind root (`inner` skipped, not counted, nothing planned under it);
   - all of them in the environment group.

**PR B, the guard and the Trash** (clean 7, journal 7, progress 5):
1. The guard and the dry run open the parent with `open_beneath`: `EXDEV` or `ELOOP` gives `parent-changed`. `EntryStat.mount_root` gives `incomplete`.
2. The Trash port.
   - `Trasher::trash(&TrashTarget { path, parent, name, id, is_dir })` returns `TrashError::{CrossesFilesystem, Other}`.
   - Updated: `SystemTrash`, the fakes, the session companions (through their parent), and `exec::trash`, which maps `CrossesFilesystem` to `Skipped(TrashCrossesFilesystem)`.
3. `platform/linux/trash.rs` becomes ddc's own move, as the amendment lists it: home trash or `$topdir` trash, info first, `renameat2(NOREPLACE)`, the `(dev, ino)` check, and nothing moved on failure. `chrono` is added for Linux, for the local `DeletionDate` (already in the lock file).
4. The dry run predicts the refusal.
5. Wording names `--permanent`, in `prompt.rs`, `CleanWording.swift` and `AfterClean` (`.never`). The schemas, DDCKit and fixtures move.
6. Tests:
   - unit tests in `linux/trash.rs` with a temporary topdir: the `.trashinfo` text, a name taken, sticky `.Trash`, a symlinked `.Trash-$uid`, info removed on failure, `NAME_MAX`;
   - unit tests below the trash folder: `files/` as a symlink or another user's is refused; the dry run and the clean choose the same way;
   - on the CI mounts: `Tmpfs` (moved into `.Trash-$uid`, `(dev, ino)` kept), `TmpfsRootTop` (refused, source intact), `BtrfsSubvolume` (refused), `Nfs` (moved into the export's `.Trash-$uid` with the plain `renameat`), and a guard on `Bind` (stale plan refused). The dry run predicts each refusal;
   - the journal line names where each item went;
   - the old crate test that asserted the copy fallback is rewritten.

7. The owner-checks runbook: the line that says an item lands in the home Trash by a copy becomes the refusal, and a real autofs map under a person's home is added.
8. `trash` is removed from the Linux target of `ddc-core` (it stays for macOS, and DDC-008's Windows work may use it).

**Close:** evidence from a green `linux-mounts` run on main.

## Acceptance Criteria

- [ ] A bind mount of the same filesystem inside a scanned root is reported as skipped, is never counted, and nothing inside it is ever planned or deleted (test on a real bind mount in the CI mounts job).
- [ ] An automount point is never triggered or opened, and the scan reports it as `automount`: an autofs direct map, found as a mount root of type `autofs`, and a folder carrying `STATX_ATTR_AUTOMOUNT` (an NFS or SMB referral). Evidence: a test on a real autofs direct map in the CI mounts job, where mountinfo gains no mount, and a unit test of `boundary()`. A real map under a person's home is an owner check.
- [ ] No clean copies a file into the Trash. A move uses the home trash on its own mount and device, or that filesystem's `.Trash-$uid`, or refuses with `trash-crosses-filesystem`; it works on NFS, which refuses rename flags; the dry run predicts each refusal (tests in the CI mounts job: tmpfs, root-topped tmpfs, btrfs subvolume, NFS).
- [ ] The safety ADR §5 states the Linux mount rules and the cross-filesystem Trash rule, critic-approved.

## Final Summary

## Technical Notes

- **Kernel facts found at planning** (fs/namei.c, fs/stat.c):
  - `RESOLVE_NO_XDEV` refuses only where the walk changes `vfsmount`, so a btrfs subvolume reached by path passes;
  - on a kernel without commit 042a60680de4 it runs `d_manage` and `d_automount` before refusing, so it triggers an automount;
  - `STATX_ATTR_AUTOMOUNT` comes from the inode's `S_AUTOMOUNT`, which NFS and SMB referrals, AFS mount points and FUSE submounts (virtiofs) set, never autofs;
  - `fstatat` always behaves as `AT_NO_AUTOMOUNT`.
- **A known gap left as it is.** `exec/delete.rs` removes a tombstone with `std::fs::remove_dir_all` by path, which would descend into a mount made after the guard's check. It was already so on macOS. It moved to story "Linux file kinds: special files, immutable flags, sparse files and odd names".
- **Considered and left:** finding `$topdir` by climbing `..` from the parent's descriptor until `STATX_ATTR_MOUNT_ROOT` (no mountinfo needed). Both work from 5.8; the climb is the simpler choice if mountinfo turns out awkward, and the plan may take it. Opening each folder from its parent's descriptor rather than from the root's would cost one descriptor per pending task; the root's is enough.
- **Accepted:** a folder of mode 0555 (a Go module cache) cannot be renamed into another parent, because the kernel rewrites its `..`. Its Trash move is `failed` with `EACCES`; a clean with `--permanent` removes it, as the delete path adds `u+rwx` after the tombstone.
- **Not this story:** a hard NFS mount whose server is dead can still hang a `statx` on its root. Story "Network and encrypted homes on Linux: NFS, CIFS, fscrypt and eCryptfs" owns it.
- `du -x` enters bind mounts of the same filesystem. Here the walk differs from `du` on purpose. The `du -skx` comparison tests need a note where a bind mount is present.
- WSL's `/mnt/c` is a 9p or drvfs mount. The walk from the home never enters it.

## Dependencies

- Story "Linux CI mounts: a filesystem matrix job with root-only steps".

## Attachments

-

---

*Last updated: 2026-11-01*

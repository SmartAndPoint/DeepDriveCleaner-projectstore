---
type: story
id: "story-linux-mount-boundaries-automounts-bind-mounts-and-the-trash-across-filesystems"
epic: "DDC-010"
title: "Linux mount boundaries: automounts, bind mounts and the Trash across filesystems"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-02
external_refs: {}
tags: [linux, scanner, safety, clean, filesystems]
code_refs: []
specs: []
started_at: "2026-11-01T18:37:21.133Z"
closed_at: "2026-11-02T09:02:22.859Z"
plan_updated_at: "2026-11-02T06:00:00.000Z"
---

# Linux mount boundaries: automounts, bind mounts and the Trash across filesystems

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | done |
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

**PR B, the guard and the Trash** (clean 7, journal 7, progress 5). Re-planned 2026-11-02 with the planner, from origin/main at 0.17.6, and split in two PRs, each with its own bump. PR A left the guard's re-walk on the anchored walker already (`walk_with_reader`, `read_in`); what is left is the guard's own opens and stats, and the Trash.

*PR B1, the guard's boundary and the Trash port, with no change on macOS:*
1. The mount id without `STATX_MNT_ID` (amendment, 2026-11-02). PR A's `anchor()` and `list_in()` keep no mount id when `statx` gives none, so their by-path fallback checks no mount there. `bulk` gains a `mnt_id(fd)` that reads `/proc/self/fdinfo/<fd>` in that case, and the guard's open checks that the root's path still names the anchored folder. The mount-root test that compares the fd's id with its `..`'s (a folder whose `..` is itself is the root, and a mount root) moved to B2, step 8, where the Trash needs it: in B1 the by-path open compares mount ids with the root's.
2. `linux/fsops.rs`: `DirFd::open_beneath(anchor, dir)`, built on `bulk`'s `openat2`, `retry` and the by-path fallback's mount check. On macOS and Windows: the path must start with the anchor's, then today's open (Windows also checks `dev`). Pinned in `platform/mod.rs`.
3. `EntryStat.mount_root`: on Linux `stat_child` uses `bulk::stat` with `AT_NO_AUTOMOUNT`, true on `STATX_ATTR_MOUNT_ROOT` or a mount id other than the parent's; false on macOS and Windows. Without a mount id it is false: the amendment accepts that case.
4. `exec/guard.rs` opens one anchor per root, and a root whose anchor cannot be opened makes every item under it `parent-changed`, never a by-path walk as the scan does; the parent opens with `open_beneath`, and any error is `parent-changed`. `mount_root` gives `incomplete` in `check()`, and counts as another volume in `check_entry` and for the session companions. `exec/preflight.rs` does the same with the guard's anchors, in place of its own `DirFd::open` of the parent.
5. The port: `Trasher::trash(&TrashTarget { path, parent, name, id, is_dir }) -> Result<Option<TrashPlace>, TrashError>` and `would_trash` (default `Ok`). `TrashError::{CrossesFilesystem, Other}`. `SystemTrash`, the fakes, the harness and loadgen trashers, `exec::trash` and `trash_session` move to it. Until B2, `CrossesFilesystem` is `failed`.
6. Tests: `open_beneath` refuses a path outside the anchor, a symlink on the way with `openat2` (by path, only the last component, as before), and a root replaced since the anchor; on the CI `Bind` mount, `stat_child` reports `mount_root` and a stale plan into the bind mount is refused; the fdinfo mount id equals `statx`'s where both exist.

*PR B2, the Linux move, the schemas and the wording:*
7. `platform::folders::data_home` reads `XDG_DATA_HOME` (a relative one ignored) into `Env`; `SystemTrash::new(&env)`.
8. `platform/linux/trash.rs` becomes ddc's own move, as the amendment lists it, through one selection used by the move and by `would_trash`: home trash or `$topdir` trash, info first, `renameat2(NOREPLACE)` from the parent's fd into `files/` (a new `rename_into`; the `EINVAL` fallback works as in `fsops::rename_child`), the `(dev, ino)` check with the rename back, and nothing left on failure. The mount ids come from step 1 when `statx` gives none; mountinfo is read at the move, never kept from the scan; `/proc` with no answer is `failed`. `statfs::point_by_id` finds `$topdir`. `chrono` for `DeletionDate`. The `trash` crate leaves the Linux target.
9. `SkipReason::TrashCrossesFilesystem`; the journal entry and each moved companion record their `TrashPlace`. `Outcome::Trashed` stays as it is.
10. The dry run keeps the guard's `Verified` and asks `would_trash` for every `WouldTrash` prediction, the native action's Trash fallback included; `PreflightPorts` gains the trasher.
11. Clean 7, journal 7, progress 5; the regenerated schemas; wording naming `--permanent` in `prompt.rs`, `CleanWording.swift` and `AfterClean` (`.never`); DDCKit and its fixtures, as commit 18533a1 moved them.
12. Tests:
    - unit tests in `linux/trash.rs` with a temporary home: the `.trashinfo` text, a name taken, sticky `.Trash`, a symlinked `.Trash-$uid`, info removed on failure, `NAME_MAX`, `files/` as a symlink or another user's;
    - the dry run and the clean choose the same way;
    - on the CI mounts (the `Bind` guard moved to B1): `Tmpfs` (moved into `.Trash-$uid`, `(dev, ino)` kept), `TmpfsRootTop` (refused, source intact), `BtrfsSubvolume` (refused), `Nfs` (moved, with the plain `renameat`). The dry run predicts each refusal;
    - the journal names where each item went;
    - the old crate test that asserted the copy fallback is rewritten.
13. The owner-checks runbook: the line that says an item lands in the home Trash by a copy becomes the refusal, and a real autofs map under a person's home is added.

Watch: story "Clean: --agent bounded consent" also moves the clean and journal schemas; whichever merges second takes the next number.

**Close:** evidence from a green `linux-mounts` run on main.

## Acceptance Criteria

- [x] A bind mount of the same filesystem inside a scanned root is reported as skipped, is never counted, and nothing inside it is ever planned or deleted (test on a real bind mount in the CI mounts job). — evidence: crates/ddc-core/tests/mount_boundaries.rs `a_bind_mount_inside_a_root_is_skipped_never_counted_and_never_claimed`, `a_bind_mount_is_refused_beneath_its_root_with_openat2_and_without`, `the_guards_stat_tells_a_bind_mount_as_a_mount_root`, `a_stale_plan_into_a_bind_mount_is_refused_and_nothing_moves`; CI run 41416834167, job linux-mounts, on main 2cba4cf
- [x] An automount point is never triggered or opened, and the scan reports it as `automount`: an autofs direct map, found as a mount root of type `autofs`, and a folder carrying `STATX_ATTR_AUTOMOUNT` (an NFS or SMB referral). Evidence: a test on a real autofs direct map in the CI mounts job, where mountinfo gains no mount, and a unit test of `boundary()`. A real map under a person's home is an owner check. — evidence: mount_boundaries.rs `an_autofs_direct_map_is_reported_and_never_triggered_by_a_scan`; platform/linux/bulk.rs `an_automount_point_or_another_mount_is_the_edge_of_the_walk`; CI run 41416834167, job linux-mounts, on main 2cba4cf; the owner check is in ops/owner-checks-manual-acceptance-across-stories.md, "Linux mount boundaries (DDC-010)"
- [x] No clean copies a file into the Trash. A move uses the home trash on its own mount and device, or that filesystem's `.Trash-$uid`, or refuses with `trash-crosses-filesystem`; it works on NFS, which refuses rename flags; the dry run predicts each refusal (tests in the CI mounts job: tmpfs, root-topped tmpfs, btrfs subvolume, NFS). — evidence: platform/linux/trash.rs unit tests; mount_boundaries.rs trash tests on `Tmpfs`, `TmpfsRootTop`, `BtrfsSubvolume`, `Nfs` with their dry runs; tests/clean_items.rs `the_dry_run_predicts_a_trash_that_would_cross`; tests/harness_clean.rs `a_session_entry_without_a_trash_is_predicted_and_skipped_as_crossing`; CI run 41416834167, job linux-mounts, on main 2cba4cf; the `trash` crate is off the Linux target (crates/ddc-core/Cargo.toml)
- [x] The safety ADR §5 states the Linux mount rules and the cross-filesystem Trash rule, critic-approved. — evidence: adr/cleanup-safety-model-risk-classes-plan-and-confirmation.md, amendment of 2026-11-01 and its 2026-11-02 paragraph on mount ids without `STATX_MNT_ID`; critic passes 2026-11-01 and 2026-11-02

## Final Summary

**What changed.** Three code PRs, each with its own release version:
- **PR A (#79, 0.17.0), the walk.** Each entry is stat'ed with `AT_NO_AUTOMOUNT` and `STATX_MNT_ID`. An automount point is skipped as the new reason `automount` (scan schema 5), and a mount root or another mount id as `other-filesystem`. Folders open beneath the scan root's descriptor with `openat2(RESOLVE_BENEATH | NO_XDEV | NO_SYMLINKS)`, or by path with a mount check.
- **PR B1 (#87, 0.18.1), the guard.** The clean and the dry run anchor each root once and open every parent beneath it. A mount crossed or a symlink on the way is `parent-changed`, and a mount at the item is `incomplete`. A root replaced since the anchor is refused. Where `statx` gives no mount id, `/proc/self/fdinfo` does. The `Trasher` port takes the verified folder and name.
- **PR B2 (#91, 0.20.0), the Trash.** ddc's own freedesktop move on Linux, never a copy:
  - home trash on the same mount and device, else the mount top's `.Trash/$uid` or `.Trash-$uid`;
  - info file first, `renameat2(NOREPLACE)` with the NFS fallback, the identity checked after the move;
  - `trash-crosses-filesystem` when no Trash is usable, predicted by the dry run;
  - the journal records the place;
  - clean 8, journal 8, progress 6.

**Why.** A bind mount of the same filesystem passed the device check, so the scan counted it and a clean could act through it. A stat could trigger an automount. And the `trash` crate copied an item across filesystems into the home.

**Tests executed.**
- Unit and integration tests on macOS CI and on a Linux machine: 869/869 at B2.
- The Gate's `linux-mounts` job on real mounts: bind, autofs, tmpfs, root-topped tmpfs, btrfs subvolume, NFS. Green on PRs #79, #87, #91 and on main (run 41416834167).

**Risks and follow-ups.**
- `exec/delete.rs`'s by-path `remove_dir_all` moved to story "Linux file kinds…".
- A symlinked `~/.local/share/Trash` refuses every move on the home mount (conservative, accepted).
- Main's run 41416834167 failed only `fuzz-lock`: B2's `chrono` dependency was missing from `fuzz/Cargo.lock`. The 0.21.0 release PR refreshes it.
- The owner checks are in the runbook.

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
- **Deviation, PR A:** a folder whose whole path reaches Linux's `PATH_MAX` (4096) is unreadable (`ENAMETOOLONG`), as by path, so its finding is incomplete. Story "Linux file kinds…" takes such paths on.
- **Follow-up:** Rust std's `symlink_metadata` uses `statx` without `AT_NO_AUTOMOUNT`, so any std stat of an automount trigger mounts it. The walk and the guard avoid std there; the test helpers read mountinfo instead.
- **Deviations, PR B2:** `XDG_DATA_HOME` is read at call time inside `platform/linux/trash.rs`, not through an `Env` field. A trash folder must also be on the item's `dev`, so the dry run predicts the btrfs-subvolume refusal. Name collisions get `.1`, `.2`, … appended. `EROFS` is `failed`, not crossing.
- **Evidence runs:** linux-mounts green on PR #79, #87 and #91, and on main 2cba4cf (run 41416834167).

## Dependencies

- Story "Linux CI mounts: a filesystem matrix job with root-only steps".

## Attachments

-

---

*Last updated: 2026-11-02*

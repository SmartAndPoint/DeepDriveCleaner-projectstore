---
type: story
id: "story-linux-ci-mounts-a-filesystem-matrix-job-with-root-only-steps"
epic: "DDC-010"
title: "Linux CI mounts: a filesystem matrix job with root-only steps"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [linux, tests, ci, filesystems]
code_refs: ["scripts/ci-mounts.sh", "scripts/native-skips.sh", "scripts/gate-scope.sh", ".github/workflows/gate.yml", ".github/workflows/nightly.yml", ".config/native-skips/linux.txt", "crates/ddc-core/tests/common/mounts.rs", "crates/ddc-core/tests/common/mod.rs", "crates/ddc-core/tests/mounts.rs", "scripts/tests/test_native_skips.py", "scripts/tests/test_gate_scope.py"]
specs: []
started_at: "2026-11-01T17:34:56.771Z"
closed_at: "2026-11-01T18:06:53.477Z"
plan_updated_at: "2026-11-01T17:34:56.771Z"
---

# Linux CI mounts: a filesystem matrix job with root-only steps

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | done |
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

- [x] Spike first, in a throwaway workflow on a branch, and record the answers in this story:
  - loop devices and `mkfs.btrfs`, `mkfs.xfs` on `ubuntu-24.04`;
  - `zfsutils-linux` and a file-backed pool;
  - `nfs-kernel-server` with a loopback export, including `all_squash`;
  - `mknod` and `chattr +i` under `sudo`;
  - an ext4 image mounted with `-O casefold` and a folder with `chattr +F`;
  - an exFAT image.
  
  Each one that does not work on the hosted runner becomes an owner check, with the reason. — done: runs 41330207570 and 41330430657 on `spike/ci-mounts` (branch deleted); every probe worked, CIFS added, no owner check (Technical Notes)
- [x] `scripts/ci-mounts.sh`. It makes the mounts the spike confirmed under one folder, owned by the runner user, and exports each one's path:
  - btrfs with a subvolume, a read-only snapshot and a writable snapshot;
  - XFS;
  - tmpfs, twice: one owned by the runner user, and one whose top is owned by root, with a user-owned folder inside it for the cross-filesystem Trash refusal;
  - ZFS with a child dataset and a snapshot;
  - loopback NFS, plain and `all_squash`;
  - casefold ext4;
  - exFAT;
  - a bind mount of the runner's ext4 inside a test root.
  
  The exported variables are `DDC_MOUNT_BTRFS`, `DDC_MOUNT_XFS` and so on. It also prepares the root-only fixtures that tests read by path from variables: a FIFO, a device node, an immutable file and an append-only file. — done: PR #71 (0.15.2), plus loopback CIFS through Samba (`DDC_MOUNT_CIFS`). No FIFO fixture: making one needs no root, so a test makes its own. CI only: the script edits `/etc/exports`, Samba and ZFS with nothing to undo it, so it refuses without `CI=true`
- [x] A test helper `common::mount(kind)`. It returns the path from the variable, or skips with the reason `no <kind> mount`. Native runs and macOS skip it. — done: `common::mount(test, Mount::…)`, `common::prepared(test, Prepared::…)` and `TreeBuilder::new_in` in `crates/ddc-core/tests/common/`; a set variable whose `f_type` is wrong fails the test; Linux only (compiled out elsewhere)
- [x] `.config/native-skips/linux.txt` gains a group "environment: needs the CI mounts". `scripts/native-skips.sh` accepts a name of that group as skipped in native runs and refuses it as skipped in the mounts job. — done: `# [environment]` header; `native-skips.sh` takes a "no … mount" skip only from that group, `--filter` and `--mounts` modes
- [x] The job `linux-mounts` in `gate.yml`:
  - runs the tests that use `common::mount`, through a nextest filter;
  - runs on main, nightly, and on a branch when `crates/ddc-core/src/platform/linux/`, `scan/`, `exec/` or the script changed (`scripts/gate-scope.sh`).
  
  Not a required check until it has been green for a week. — done: `gate-scope.sh mounts` (wider than listed: any test file, the manifests, the skip lists, the workflows) and a twin job in `nightly.yml`
- [ ] The native-runs runbook: an optional local step for a Linux machine where loop mounts are allowed. — moved to story "Linux test parity: goldens, pinned trees, distro smoke and a grouped skip list": it needs `native-linux.sh` to pass the mounts into its container, and a script that can undo what it sets up

## Implementation Plan

Planned 2026-11-01 with the planner, from origin/main c344655. One code PR (0.14.11).

1. **Spike** (branch `spike/ci-mounts`, never merged): one throwaway workflow on `ubuntu-24.04`, each probe its own step. Results under Technical Notes.
2. **`scripts/ci-mounts.sh up <base>`**: every mount the spike confirmed, under `/tmp/ddc-mounts`, outside `$HOME` (`RUNNER_TEMP` is inside it, and the test trees' guard refuses that). Each privileged step goes through `sudo -n`. It refuses outside CI without `--this-machine`. It prints `DDC_MOUNT_*` and `DDC_FIXTURE_*` lines for `$GITHUB_ENV`, and a table of filesystem types.
3. **`crates/ddc-core/tests/common/mounts.rs`**: `common::mount(test, Mount::…)` and `common::prepared(test, Prepared::…)`, plus `TreeBuilder::new_in(dir)`. An unset variable skips with "no <kind> mount (VAR unset)". A set variable whose `f_type` is wrong fails the test.
4. **`crates/ddc-core/tests/mounts.rs`**: Linux only. One sanity test per mount and fixture, checking both `f_type` and the product's own `platform::statfs` (mountinfo type, mount point, source).
5. **Skip groups.** `.config/native-skips/linux.txt` gets `# [environment]`. `scripts/native-skips.sh` takes a "no … mount" skip only from that group. `--filter` prints the group's nextest filter. `--mounts` fails on any skip, and on any group test that did not pass.
6. **`scripts/gate-scope.sh mounts <sha>`** answers `yes` when the change since the merge base touches what the mounts judge, and on every doubt.
7. **The `linux-mounts` job in `gate.yml`.** Its steps are gated on the scope, never the job itself. It has `continue-on-error` while it is not required. A twin job runs in `nightly.yml`.
8. **`AGENTS.md`**: how a red, non-required check interacts with `--fail-fast`.

Deferred: the local mounts step on the owner's machine. It needs `native-linux.sh` to pass the mounts into its container (`-v <base>`, `--env-file`), which is a follow-up. Until then, the environment group runs in CI only.

## Acceptance Criteria

- [x] The spike's answers are recorded here: which mount types work on the hosted runner, and which became owner checks. — evidence: Technical Notes, "Spike results" (runs 41330207570 and 41330430657): every type worked, none became an owner check
- [x] The `linux-mounts` job is green on main and names each mount in its log. One test per mount type proves that the mount is really that filesystem (`statfs` `f_type`). — evidence: Gate run 41334010096 on main 02ddbd8, every job green. The `linux-mounts` step "Mounts" prints the table of each mount and its type, and `native-skips.sh --mounts` printed "the 17 tests of the environment group passed on the mounts". `common::mount` asserts the `f_type` of every mount kind (`crates/ddc-core/tests/common/mounts.rs`), and `crates/ddc-core/tests/mounts.rs` holds one test per mount and fixture, which also checks the product's own `platform::statfs`
- [x] A test that uses `common::mount` skips with its reason in native runs, and `scripts/native-skips.sh` accepts this only for the environment group (test of the script). — evidence: the `check-x86_64-unknown-linux-gnu` job and a native Linux run log "skipped a_btrfs_mount_is_btrfs_in_mountinfo: no btrfs mount (DDC_MOUNT_BTRFS unset)", and native-skips accepted 138 listed skips (762 tests). `scripts/tests/test_native_skips.py` class `Groups`: `test_a_mount_skip_outside_the_environment_group_fails`, `test_another_group_ends_the_environment_group`, `test_an_environment_test_skipped_for_another_reason_fails` and `test_the_mounts_run_fails_on_any_skip`

## Final Summary

**What changed (PR #71, 0.15.2).**
- `scripts/ci-mounts.sh` mounts these on the Gate's Linux runner itself:
  - btrfs (a subvolume, read-only and writable snapshots), XFS, and ZFS (a child dataset and a snapshot);
  - NFS (plain and `all_squash`) and CIFS;
  - casefold ext4 and exFAT;
  - tmpfs, twice (one with a root-owned top);
  - a bind mount.
  
  It also prepares a device node, an immutable file and an append-only file. It runs on CI only.
- `common::mount` and `common::prepared` read the mounts. They skip with "no <kind> mount" elsewhere, and fail when a mount is not the filesystem it claims.
- 17 sanity tests prove each mount and fixture.
- The skip list's `# [environment]` group is the only place such a skip is accepted. The `linux-mounts` job runs that group, and any skip fails it.
- `gate-scope.sh mounts` decides when a branch runs the job, and a failed scope runs it.
- A twin job runs nightly.
- The job is not a required check yet.

**Why.** The safety stories of this epic can prove their criteria only on real btrfs, ZFS, NFS, bind mounts and root-made files, and a story is done only with evidence.

**Tests executed.**
- Gate run 41334010096 on main: every job green, including `linux-mounts` with 17/17 passed.
- Branch runs 41331798408 and the PR's run.
- A native Linux run: 762 passed, 138 listed skips.
- `scripts/tests` for native-skips and gate-scope: 70 OK.

**Risks and follow-ups.**
- Make `linux-mounts` a required check after a week of green runs. That needs the ruleset (the owner's), `BASE_JOBS` and `scripts/ci-green.sh`, and removing `continue-on-error`.
- The local mounts step on the owner's machine moved to story "Linux test parity: goldens, pinned trees, distro smoke and a grouped skip list".
- `native-skips.sh` finds an environment skip by its reason text ("no … mount" or "no … fixture"). Another test's reason must not start that way.

## Technical Notes

- **Spike results** (2026-11-01, `ubuntu-24.04` image 20261025, kernel 7.0 azure, passwordless `sudo`). Every probe worked:
  - loop btrfs: the user creates subvolumes and both kinds of snapshot without root, and `cp --reflink=always` works. `btrfs subvolume show` as the user fails ("Could not search B-tree"), so the btrfs story must use `GET_SUBVOL_INFO` itself;
  - XFS (at least 300 MB), with reflink;
  - ZFS: `zfsutils-linux`, `modprobe zfs`, a pool on a file, a child dataset and a snapshot reachable at `.zfs/snapshot`;
  - NFS 4.2 on loopback: plain, and `all_squash`, where new files belong to 65534;
  - CIFS through a Samba guest share: case-insensitive, `f_type` SMB2;
  - casefold ext4, with `chattr +F` done by the user;
  - exFAT: case-insensitive;
  - tmpfs, including a root-owned top;
  - a bind mount;
  - `sudo mknod`, `sudo chattr +i/+a`.
  
  Installing the packages takes about 20 s. `/`, `/tmp`, `/mnt` and `$HOME` are one ext4 filesystem. No mount type became an owner check. CIFS joined the list for story "Network and encrypted homes on Linux: NFS, CIFS, fscrypt and eCryptfs".
- `chattr -i` is root-only too. The prepared immutable file stays on the runner, which is discarded after the job.
- Keep the mounts small (a few hundred MB). The job must finish in minutes.

## Dependencies

- None. The mount, btrfs and ZFS, network-home, file-kind and private-size stories depend on this one.

## Attachments

-

---

*Last updated: 2026-11-01*

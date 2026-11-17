---
type: story
id: "story-network-and-encrypted-homes-on-linux-nfs-cifs-fscrypt-and-ecryptfs"
epic: "DDC-010"
title: "Network and encrypted homes on Linux: NFS, CIFS, fscrypt and eCryptfs"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-17
external_refs: {}
tags: [linux, scanner, safety, filesystems, nfs]
code_refs: [".config/native-skips/linux.txt", ".config/native-skips/windows.txt", "scripts/ci-mounts.sh", "scripts/native-linux.sh", "app/Packages/DDCKit/Sources/DDCKit/CleanWording.swift", "app/Packages/DDCKit/Sources/DDCKit/ScanReport.swift", "crates/ddc-cli/src/cmd/agent_clean.rs", "crates/ddc-cli/src/cmd/background.rs", "crates/ddc-cli/src/cmd/clean.rs", "crates/ddc-cli/src/cmd/clean_items.rs", "crates/ddc-cli/src/cmd/scan.rs", "crates/ddc-cli/src/prompt.rs", "crates/ddc-cli/src/render.rs", "crates/ddc-cli/tests/scan_cli.rs", "crates/ddc-cli/tests/structure.rs", "crates/ddc-core/src/changes/not_read.rs", "crates/ddc-core/src/exec/delete.rs", "crates/ddc-core/src/exec/guard.rs", "crates/ddc-core/src/exec/journal.rs", "crates/ddc-core/src/exec/mod.rs", "crates/ddc-core/src/exec/report.rs", "crates/ddc-core/src/holders.rs", "crates/ddc-core/src/model.rs", "crates/ddc-core/src/platform/mod.rs", "crates/ddc-core/src/platform/types.rs", "crates/ddc-core/src/platform/linux/bulk.rs", "crates/ddc-core/src/platform/linux/fsops.rs", "crates/ddc-core/src/platform/linux/meta.rs", "crates/ddc-core/src/platform/macos/fsops.rs", "crates/ddc-core/src/platform/macos/meta.rs", "crates/ddc-core/src/platform/windows/fsops.rs", "crates/ddc-core/src/platform/windows/meta.rs", "crates/ddc-core/src/report/mod.rs", "crates/ddc-core/src/report/volumes.rs", "crates/ddc-core/src/rules/catalog/linux.rs", "crates/ddc-core/src/rules/catalog/mod.rs", "crates/ddc-core/src/rules/mod.rs", "crates/ddc-core/src/scan/mod.rs", "crates/ddc-core/src/scan/walker.rs", "crates/ddc-core/src/store.rs", "crates/ddc-core/tests/after_clean.rs", "crates/ddc-core/tests/changes.rs", "crates/ddc-core/tests/clean_safety.rs", "crates/ddc-core/tests/disk_free.rs", "crates/ddc-core/tests/file_kinds.rs", "crates/ddc-core/tests/matcher_reference.rs", "crates/ddc-core/tests/network_homes.rs", "crates/ddc-core/tests/report.rs", "crates/ddc-core/tests/worktrees.rs", "docs/clean.schema.json", "docs/footprint.schema.json", "docs/progress.schema.json", "docs/scan.schema.json"]
specs: []
started_at: "2026-11-17T07:40:56.185Z"
closed_at: "2026-11-17T16:26:10.052Z"
plan_updated_at: "2026-11-17T07:40:56.185Z"
status_changed_at: "2026-11-17T16:26:10.052Z"
---

# Network and encrypted homes on Linux: NFS, CIFS, fscrypt and eCryptfs

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | done |
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

- [x] Detect a network filesystem by mountinfo's filesystem type (`nfs`, `nfs4`, `cifs`, `smb3`, `fuse.sshfs`); every FUSE mount shares one `f_type`, so mountinfo tells sshfs from gocryptfs. Only NFS can be mapped (safety ADR, 2026-11-17).
- [x] Ownership.
  - When a root is mapped (safety ADR, 2026-11-17: an `all_squash` NFS export, observed through the run's own holder file), report it once as a note on the volume ("files are owned by `<uid>`: the server maps ids"), and not as thousands of `foreign-owner` lines.
  - Nothing the user cannot delete is planned.
  - Say in the plan's text and in the clean's text report when `--permanent` may be the only working clean.
- [x] Silly renames. `.nfs*` files are never counted as new findings after a clean. The clean report names a delete that left one.
- [x] Measure a scan on the loopback NFS export of the CI mounts job, and record the latency in the research note's open question.
- [x] Encrypted homes: the eCryptfs lower folder (`~/.Private`, `/home/.ecryptfs`) is never walked. Add `~/.Private` to the Linux protected folders through the safety ADR §6, unless it is already covered.
- [x] Tests on the CI mounts job's NFS export, plain and `all_squash`:
  - a scan finishes and reports the mapping once;
  - a clean of a selected cache either removes it or refuses it with a reason, and leaves no tombstone except a folder that holds a silly rename, named as `left_open`.

## Implementation Plan

Planned 2026-11-17 with the planner, from origin/main e276596 (after 0.36.0). The contract is the safety ADR's amendment of 2026-11-17 for this story, critic-reviewed; as for the three earlier DDC-010 safety stories, no separate spec. Current schema versions: scan 9, clean 13, journal 14, progress 11.

Decided by the owner on 2026-11-17: on a mapped root the mapped owner's files count as the user's (option b). Taken from the planner's recommendations: the left `.nfs*` files are an optional `left_open` list, not a new outcome; fscrypt and a dead hard mount are text and owner checks only.

Already in place:
- the CI mounts job exports NFS plain (`DDC_MOUNT_NFS`) and with `all_squash,anonuid=65534` (`DDC_MOUNT_NFS_SQUASH`), and mounts CIFS with `uid=` (`DDC_MOUNT_CIFS`); `common::Mount::{Nfs,NfsSquash,Cifs}` exist;
- `foreign-owner` per finding: the walker's `OwnerAcc::foreign`, `report/mod.rs::finding()`, the guard's refusal from `not_selected`, and `doomed()` adding `u+rwx` only when `st.uid == me`;
- the Trash's ownership and mode checks in `platform/linux/trash.rs`, so a mapped home's Trash already refuses as `trash-crosses-filesystem`.

Five code PRs, no version bump (batched releases), each merged only after the `linux-mounts` job itself is green (it is `continue-on-error` and not required):

1. **eCryptfs folders (no schema move).** `rules/catalog/linux.rs`: `.Private` and `.ecryptfs` in `OWN_PROTECTED_IN_HOME`, `/home/.ecryptfs` in `NEVER_WALKED`, and a home-relative never-walked list inserted in `rules/mod.rs::resolved()` as `no_traverse`. Tests: the catalog lists on every OS; on Linux a real `.Private` with a decoy `node_modules` never opened (a counting reader) and skipped as `excluded`, and a plan naming it refused as `protected`. Closes AC4.
2. **NFS latency (script only).** `scripts/ci-mounts.sh`: a tree of about 20k files on the NFS export and on local disk, `ddc scan` cold after a remount and warm, entries and µs per entry to the step summary, `stx_dev` before and after the remount. Then a vault PR records it in the research note. Closes AC3.
3. **Detection and the id-mapping note.** A classifier of mountinfo types in the three `platform/*/statfs.rs`, pinned in `platform/mod.rs` (`is_network_fs` keeps its meaning). The probe: the run's own holder file in `<data-dir>/lock.holders/` (`holders.rs`), its mount id against the root's and its owner as the server shows it; a neutral platform call reads both. Foreign-uid counts of M in the walker, out through `OwnerResult`/`RootResult`; the decision as a pure function in `report/volumes.rs`, called from `walk_classified`, so the scan, `--items` (`report/recheck.rs`) and `ddc agent clean` (`recheck::scoped`) each decide from their own probe; optional `Volume.id_mapping` in `model.rs`; one line in `render.rs`; `docs/scan.schema.json` regenerated. Findings stay `foreign-owner` here. Closes AC1 and, with PR 4, AC5. Tests: `tests/report.rs` fakes (one note; none for a holder file of the user's, a data dir on another mount, M = 0, M = 4294967294, no holder record, CIFS, `fuse.sshfs`, a root outside the home); a `render.rs` unit test; new `tests/network_homes.rs` on the mounts with `--data-dir` on the export (`NfsSquash` one note with 65534; with a local data dir none; `Nfs` and `Cifs` none).
4. **Planning on a mapped root.** From PR 3's review: decide each root before the walk (its type and `root_access` are known then) and give the walk and `finding()` a per-root M, so M's entries never pass on a root that is not mapped (CIFS, outside the home, another mount); count M's entries per owner too (`OwnerResult`), and the claimed folder's own uid, so a third uid is `foreign - mapped`; build `id_mappings` from the same decision. AC1's "no `foreign-owner` line" closes here, not in PR 3. `finding()` treats M's entries as the user's when the mapping holds; `platform::meta::writable_dir` (`faccessat` with `AT_EACCESS`) in three modules; `Finding.mapped_owner` for the report and the app, and M from the clean's own probe into the guard's `Verified` and into `DirFd::remove_tree` and `doomed()`; one `--permanent` line in `prompt.rs` and in the clean's text report. Tests: `clean_safety.rs` fakes (a mapped finding selected, a third uid refused before any rename, a local type not mapped, a clean whose probe gives another M refusing the scan's M's entries); on `NfsSquash` a `.cache/pip` with a 0555 subfolder: with a Trash made through the export the dry run predicts `trash-crosses-filesystem`, without one the clean refuses it; nothing moves; a permanent clean removes it with no tombstone.
5. **Silly renames (scan 10).** `bulk.rs::remove_tree`, after the last `rmdir`s meet `ENOTEMPTY` on NFS, lists those folders again and tries only `.nfs` + 24 hex names (`EBUSY` leaves one, success removes an orphan, any other name keeps the item `failed`); with only `EBUSY` files left it returns a `LeftOpen { paths }` payload (`platform/types.rs`, told by `LeftOpen::of`); `exec/delete.rs` reports the item deleted with optional `left_open` on `CleanItem` and the journal item line (paths through `json_path`); the walker counts such files in `files`, `bytes` and a new per-owner `held_open`, lists them as the new scan `SkipReason::HeldOpen` without making the owner `incomplete`, gives the finding the new `NotSelected::HeldOpen`, and drops an owner whose every file is one (`files == held_open > 0`); the guard refuses it as the clean's existing `held-open` before its `incomplete` check, and that reason's doc comment in `exec/mod.rs` stops saying Windows only; and the visitor keeps them out of large files. The bump set as in commit a4d4902: `ScanReport.swift`, the DDCKit scan fixtures, the goldens' version line, `fuzz/regressions/last_scan`, `docs/*.schema.json`, the `model.rs` history line. Tests: the name matcher and the "only silly names left" decision as Linux unit tests; on `Nfs` a file held open in a selected cache, a permanent clean with no process list: deleted, `left_open` names the `.nfs` file; with the real process list refused `in-use`; a rescan while open has no finding there; after close the clean leaves nothing. Closes AC2's second half.

Every new mounts test goes into the `[environment]` group of `.config/native-skips/linux.txt`. Iteration runs on the Windows host's WSL through `scripts/native-linux.sh`; `ci-mounts.sh` runs only in CI. Last, in the vault: the research note, the owner-checks runbook (a real NFS home with an id mismatch, an Ubuntu eCryptfs home, fscrypt, a dead hard mount), the story close and the epic's `code_refs`.

## Acceptance Criteria

- [x] On a loopback NFS export with `all_squash`, a scan finishes and reports the id mapping once, not per entry (test in the CI mounts job). — evidence: `crates/ddc-core/tests/network_homes.rs` `a_squashed_nfs_home_is_reported_once_as_mapped` (linux-mounts green on #165 and #166); fakes in `tests/report.rs` `a_squashed_home_is_reported_once_as_mapped_on_its_volume`; main 58a61b3, 38e880f
- [x] A clean on that export removes a selected cache or refuses it with a reason, and leaves no tombstone. On the plain export, a clean whose cache holds an open file leaves only the folder that holds the silly rename, named as `left_open`; that `.nfs*` file is not counted later, and after it is closed the next clean removes the folder (tests). — evidence: `tests/network_homes.rs` `a_squashed_home_clean_removes_a_read_only_cache_whole`, `a_squashed_exports_trash_is_refused_and_nothing_moves` (linux-mounts on #166), `a_file_held_open_on_nfs_is_left_open_in_its_tombstone_and_goes_once_closed` (linux-mounts on #168, ran and passed on the real export); `tests/report.rs` `a_silly_rename_on_nfs_holds_its_finding_and_a_tombstone_of_them_is_none`; Linux unit tests in `platform/linux/bulk.rs`; main 38e880f, f80a3ea, d0d98e8, be7860e
- [x] The research note records the measured NFS latency, without describing any owner machine. — evidence: [[research/where-developer-disk-space-goes-linux|Where a developer's disk space goes (Linux)]], open question answered from CI run 44051015005 (vault #139); `scripts/ci-mounts.sh` latency step, main abbd7ce
- [x] `~/.Private` is never walked (test). — evidence: `tests/clean_safety.rs` `ecryptfs_folders_in_the_home_are_never_walked_or_removed`, `tests/changes.rs` for `ddc changes`; main b24206d, ac861d7
- [x] On the plain NFS export and on the CIFS mount no mapping is reported, and an entry of a third uid on a mapped root stays `foreign-owner` and is refused before any rename (tests). — evidence: `tests/network_homes.rs` `a_plain_nfs_or_cifs_home_is_not_mapped` (linux-mounts); `tests/clean_safety.rs` `on_a_mapped_root_the_mapped_owners_findings_are_planned_and_a_third_uid_is_not`; main 58a61b3, 38e880f

## Final Summary

**What changed.** Five code PRs, all merged with the `linux-mounts` job itself green, no version bump (batched releases):
1. #163 (b24206d, ac861d7): eCryptfs's `~/.Private`, `~/.ecryptfs` and `/home/.ecryptfs` are never walked and protected; `ddc changes` no longer lists them as behind a prompt.
2. #164 (abbd7ce): the CI mounts job measures a scan of about 20k entries on the NFS export and on local disk, cold and warm; the numbers went into the research note (vault #139).
3. #165 (58a61b3, 263a78a, f16d770): the scan reads the server's owner of this run's own holder record; a home on an NFS root mapped to that owner gets one `id_mapping` note on its volume.
4. #166 (38e880f): on a mapped root the mapped owner's files count as the user's (the owner's decision), decided per root before the walk; they are planned, made writable on their way out, and a third uid stays `foreign-owner`; the scan's text says the Trash is not private there.
5. #168 (f80a3ea, d0d98e8, be7860e): an NFS silly rename left by a delete is reported as `left_open` on a `deleted` item; the scan counts such files as `held-open` (scan schema 10), a tombstone of them is no finding, and a worktree holding one is held too.
Also #162 (2b126e7): `scripts/native-linux.sh` removes files git no longer tracks.

**Why.** University and corporate homes live on NFS or CIFS; before this story an `all_squash` home showed thousands of `foreign-owner` lines and removed nothing, and a delete with a file still open left a tombstone that came back as a finding.

**Tests executed.** The full native Linux leg in WSL before each push; DDCKit's 608 tests on the Mac for PR 5; the Gate on every PR, including `linux-mounts`, whose NFS (plain and `all_squash`) and CIFS tests ran against the real mounts. The silly-rename test first failed in CI because the holder was the test's own child (the process table leaves the clean's own children unread, by design); fixed in be7860e and green.

**Risks and follow-ups.**
- Manual checks are in the owner-checks runbook, section "Network and encrypted homes on Linux (DDC-010)": an `all_squash` NFS home, an NFSv4 idmap mismatch (not mapped, as the ADR says), a CIFS home, a file held open on NFS by the user and by another account, an Ubuntu eCryptfs home, fscrypt, a dead hard mount.
- Accepted in the safety ADR (2026-11-17): a dead hard mount hangs a scan as `ls` does; `ddc agent footprint` does not know NFS roots; an `EBUSY` in the first listing stays `failed`.
- Not in the ADR, a known gap: a delete that left a silly rename reports the item's whole `bytes` as freed, though the server frees the open file's space only when it is closed.
- NFSv4.1 servers that preserve unlinked files would give no silly rename; the CI kernel did rename, so the test holds there, and the product then simply deletes.

## Status log

- 2026-11-17T07:40:56.185Z status: planned → in-progress — work started (plan gate)
- 2026-11-17T16:26:10.052Z status: in-progress → done — closed (close gate)

## Technical Notes

- The Trash on a network home follows the ownership check of story "Linux mount boundaries: automounts, bind mounts and the Trash across filesystems". Under `all_squash` a Trash made through the export is `nobody`'s, so the move is refused as `trash-crosses-filesystem` and `--permanent` is the clean that works; the dry run predicts that only when the Trash already exists. A plain numeric id mismatch under NFSv3 is not mapped: its old files stay `foreign-owner`.
- Known traps (safety ADR, 2026-11-17): a dead hard mount hangs a scan in the kernel and a background scan keeps its lock; where the server refuses the chmod, a read-only folder makes the item `failed` with its tombstone named; the mapping needs the data dir on the same mount as the root, so the mounts tests pass `--data-dir` on the export; a tombstone that holds only silly renames stays until the file is closed.
- The `linux-mounts` job is `continue-on-error` and not a required check, while AC1, AC2 and AC5 rest on it: each PR merges only after that job itself is green.

## Dependencies

- Story "Linux CI mounts: a filesystem matrix job with root-only steps", for the NFS export.

## Attachments

-

---

*Last updated: 2026-11-17*

---
type: story
id: "story-synthetic-load-trees-large-wide-deep-and-hostile"
epic: "DDC-005"
title: "Synthetic load trees: large, wide, deep and hostile"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [rust, testing, performance, load]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Synthetic load trees: large, wide, deep and hostile

| Field | Value |
|---|---|
| **Epic** | [DDC-005](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

Load tests need large trees that are the same on every run and every machine. Real homes differ and change. The `aa64c83` quadratic steps showed up only on large system trees with many sibling folders. A tree of a few hundred files hides any O(n²) step.

This story adds a generator. It builds a tree from a profile, a size and a seed, always with the same shape. It writes empty or sparse files, so a tree costs inodes rather than gigabytes. M and larger trees build on a disposable disk image, so millions of entries never touch the user's volume and cleanup is one detach. S trees (10⁴ entries) go to a marked temporary folder, so the fast gate needs no image.

Profiles (each is a shape real disks have):

| Profile | Shape |
|---|---|
| `projects` | N projects with `.git`, `node_modules` (deep, many small files), `target`, `.venv`, lock files, keep markers |
| `wide` | single folders with 10⁴–2·10⁵ entries |
| `wide-root` | one project root with 10⁴+ siblings and many findings inside it, plus requirement files (`requirements.txt`, `package.json`): the shape of `aa64c83` step 2, where every finding re-read the same root |
| `deep` | paths nested to a depth of 200, with names close to `NAME_MAX` |
| `deep-path-max` | paths that reach `PATH_MAX` (1024 on macOS) and one level beyond it, separate from `deep` so a failure names the cause. The generator creates them with `mkdirat` relative to a parent handle (absolute paths fail with `ENAMETOOLONG`); the manifest says entries beyond `PATH_MAX` are reported as skipped, with no abort |
| `users` | thousands of per-user folders, each with caches and small files, the shape of `/Users`, `/private/var/folders` and system trees |
| `links` | many hard links to the same inodes, symlink loops, dangling symlinks, symlinks pointing out of the tree |
| `names` | Unicode NFC/NFD pairs, spaces, newlines and control characters, invalid UTF-8 (where the filesystem allows it), names that look like flags |
| `hostile` | unreadable folders, plus a mutator thread that deletes, renames and creates entries while a scan runs |
| `system-root` | a fake data volume root (`Users`, `Library`, `private/var/folders`, `Applications`) for `ddc disk --system-root` |
| `mixed` | all of the above, weighted like the reference machine's home |

Size tiers: S = 10⁴ entries (fast gate), M = 10⁵ (CI), L = 10⁶ (nightly and manual), XL = 10⁷ (manual).

## Decomposition

- [ ] A workspace tool crate `ddc-loadgen` (`publish = false`, not shipped), with a CLI: `ddc-loadgen <profile> --entries N --seed S --into <dir>`. It prints a manifest:
  - entry counts;
  - expected findings per rule and expected totals;
  - expected plan items for `clean --dry-run` with the default settings.
- [ ] Each profile is its own module. The manifest is computed from the same description that builds the tree, not from running `ddc`.
- [ ] Home. Most catalog rules are anchored at the home folder (`Catalog::builtin(home)`). Each profile's description names the home its manifest assumes: none (content-matched project rules only) or a fake home inside the tree (`<tree>/home`, for `users`, `mixed` and `system-root`). The harness sets `HOME` to it. Home-anchored findings are expected only when the profile provides a home. It is never the real home.
- [ ] Modification times are backdated by the description (older than the default `min_age` of 14 days where a finding is expected, newer where it is not), so the expected plan is exact.
- [ ] Safety guard, by filesystem identity, not by path prefix. Two target kinds are allowed:
  - M and above on macOS: a volume the generator mounted itself. Before the first write it calls `statfs` on the target and checks that `f_mntfromname` equals the device `hdiutil attach -plist` returned.
  - S trees (all platforms) and everything on Linux and Windows: a fresh directory under the system temporary folder (`$TMPDIR`), holding a marker file the generator created.
  
  In both cases it refuses any target on the same filesystem as the real home or `/` (compared by `statfs` device, with the real home from `getpwuid(getuid())`, not from `$HOME`), unless it is the marked temporary folder. So the fast gate needs no disk image.
- [ ] macOS `make load-image`:
  - creates a sparse APFS disk image (`hdiutil create -type SPARSE`);
  - detaches a stale image from an interrupted run first;
  - attaches it with `hdiutil attach -nobrowse -plist` at `/Volumes/ddc-load-<pid>` (no `sudo` needed);
  - writes `.metadata_never_index` and `.fseventsd/no_log` at the root, so Spotlight and fseventsd do not index millions of entries;
  - detaches and deletes it afterwards.
- [ ] Generation time per tier is recorded. L must build in ≤ 5 min on the reference machine. Otherwise the tier is cached as an image.
- [ ] A test in the fast gate: S-size trees of every profile generate identically for the same seed (the manifest hash is stable).
- [ ] Golden check. `make golden-record` and `make golden-check`: for every S profile except `hostile`, run `scan --json`, `rules --json` and `disk --json` (on `system-root`) with a fixed `HOME` and `--data-dir`. Normalise `generated_at` and `duration_ms`, store the outputs under `bench/golden/`, and diff. This is the behaviour-preservation check DDC-006's refactors and DDC-007's move and catalog split use.
- [ ] `hostile`: the mutator runs during a scan, and the scan finishes. Vanished entries are counted as skipped, not as errors that abort.

## Implementation Plan

## Acceptance Criteria

- [ ] `ddc-loadgen` builds every profile at S and M; the same seed gives the same manifest hash.
- [ ] A scan of each profile's S tree except `hostile` finds exactly the manifest's expected findings and totals, and `clean --dry-run` plans exactly its expected items (asserted in the fast gate). For `hostile`, the scan completes without a panic and its findings are a subset of the manifest's static set.
- [ ] `make golden-check` passes on an unchanged tree and fails on a seeded change to one rule (test).
- [ ] The generator refuses a folder on the home's or the boot volume's filesystem without its marker, and a mount point whose `statfs` device does not match the attached image (tested).
- [ ] An L `mixed` tree (10⁶ entries) is generated, scanned and removed on the reference machine; times are recorded in this story.
- [ ] Under `hostile`, a scan completes with vanished entries reported as skipped and no panic.

## Final Summary

## Technical Notes

- `hdiutil create`, `attach` and `detach` need no `sudo` and raise no permission prompt. Nothing here needs elevated rights.
- An APFS sparse image grows with the data written, and empty files cost metadata only. Expect about 1 KiB per entry: about 10 GB for XL. That is why XL is manual.
- Anything moved to the Trash on the image goes to the image's own `.Trashes`, not the user's Trash. It disappears with the detach.

## Dependencies

- Story "Mandatory gate before every commit".

## Attachments

-

---

*Last updated: 2026-10-28*

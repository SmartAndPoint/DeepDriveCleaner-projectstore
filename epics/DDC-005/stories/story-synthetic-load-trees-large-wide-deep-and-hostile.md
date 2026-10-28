---
type: story
id: "story-synthetic-load-trees-large-wide-deep-and-hostile"
epic: "DDC-005"
title: "Synthetic load trees: large, wide, deep and hostile"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [rust, testing, performance, load]
code_refs: []
specs: []
started_at: 2026-10-28
closed_at: null
plan_updated_at: 2026-10-28
---

# Synthetic load trees: large, wide, deep and hostile

| Field | Value |
|---|---|
| **Epic** | [DDC-005](../epic.md) |
| **Status** | in-progress |
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
| `wide-root` | one project root with 10⁴+ siblings and many findings inside it, plus requirement files (`requirements.txt`, `package.json`): the shape of `aa64c83` step 2, where every finding re-read the same root. Venvs sit directly under the root (parent = project root) with no lock file beside them, and `requirements.txt` is in the root, so `dir_has_any` reads the root once per venv |
| `deep` | one branch nested to a depth of 200 with 1–3-byte names, and a separate branch of at most 3 levels with 255-byte names (`NAME_MAX`). Both stay under `PATH_MAX` with the fixed root length below (the depth-200 branch tops out near 160 + 200·4 = 960 bytes, the long-name branch near 160 + 3·256 = 928), so `deep` reports no errors: depth 200 with long names would cross `PATH_MAX` within about 4 levels |
| `deep-path-max` | a chain of directories inside a finding (a `node_modules`) up to the `PATH_MAX` boundary, separate from `deep` so a failure names the cause. `PATH_MAX` (1024 on macOS) counts the NUL: the deepest readable directory has a 1023-byte path, and its child, at 1024 bytes or more, fails to open. The generator creates the chain with `mkdirat` relative to a parent handle (absolute paths fail with `ENAMETOOLONG`). The manifest asserts today's behaviour: that child is the one error (`ENAMETOOLONG`, counted in `errors_total`), it makes the finding `incomplete`, and nothing below it is listed. The scan does not abort, and the files listed in the last readable directory still count. A dedicated skip reason would be a schema change, outside this story |
| `users` | thousands of per-user folders, each with caches and small files, the shape of `/Users`, `/private/var/folders` and system trees |
| `links` | many hard links to the same inodes, symlink loops, dangling symlinks, symlinks pointing out of the tree |
| `names` | Unicode NFC/NFD pairs, case pairs, spaces, newlines and control characters, names that look like flags. At build time the generator probes the filesystem: normalization- and case-insensitivity, and whether it accepts invalid UTF-8 (APFS refuses it with `EILSEQ`). It records the results in the manifest's `fs_caps` and expects only the names that exist. Invalid UTF-8 is built only where the filesystem accepts it (Linux, DDC-007) |
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
  - Per-rule fixtures are derived from the catalog's public declarations: `RULES`, each rule's `Matcher`, class, lock files and removal. It follows `rule_fixture` in `ddc-core/tests/common/fixtures.rs`, but stays a copy: `ddc-core`'s tests cannot depend on the generator, which depends on `ddc-core`. A new rule without a fixture fails the generator's tests, as it fails the existing ones.
  - The expected class and selection repeat only the tree-dependent rules: app-owned and no-lockfile Review (`rules/classify.rs`), keep markers, and project activity against `min_age`. They never call `scan` or `classify`.
- [ ] Sizes are the same on every filesystem. Payload files are exactly 4096·k bytes long, with non-zero content. Marker files whose content a matcher reads (`CACHEDIR.TAG`, `CMakeCache.txt`, `pyvenv.cfg`, toolchain pins) are padded to exactly 4096 bytes with comment lines. Everything else is empty. Symlink targets stay under 60 bytes, which ext4 stores without a block (DDC-007). The tree root, after `canonicalize()`, is padded to a fixed length of 160 bytes, so path lengths, the `PATH_MAX` boundary and the order of findings are the same on every machine.
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
- [ ] A test in the fast gate: S-size trees of every profile generate identically for the same seed and the same `fs_caps` (the manifest hash is stable). Only `names` depends on `fs_caps`.
- [ ] Golden check. `make golden-record` and `make golden-check`: for every S profile except `hostile`, run `scan --json`, `rules --json` and `disk --json` (on `system-root`) with a fixed `HOME` and `--data-dir`. Normalise `generated_at`, `duration_ms`, idle days and the tree root (replaced by a placeholder, since `$TMPDIR` differs per user and machine), store the outputs under `bench/golden/`, and diff. The golden of `names` is keyed by its `fs_caps`. This is the behaviour-preservation check DDC-006's refactors and DDC-007's move and catalog split use.
- [ ] `hostile`: the mutator deletes, renames and creates filler inside a dedicated `churn/` folder while a scan runs, and the scan finishes. The filler matches no rule. A rename or a re-creation under the same name shows up as `Changed` (counted in `skipped_total`), not as vanished. An entry that vanishes between listing and reading is counted in the walk's `vanished` and makes its owning finding `incomplete`. It is not an error and does not abort the scan. A deterministic test wraps the directory reader (`scan::walk_with_reader`) to delete an entry just before it is read, and asserts that count.

## Implementation Plan

Planned with `projectstore:planner` on 2026-10-28. Three commits; the first is useful on its own.

1. **Generator and S manifests.**
   - The crate `crates/ddc-loadgen` (lib + bin):
     - a SplitMix64 RNG;
     - the description type, and the build from it over parent file descriptors, with mtimes set in a second pass;
     - the per-rule fixtures from the catalog;
     - the manifest and its FNV-1a hash over canonical JSON;
     - an in-process `verify` (`report::scan`, `plan::build_plan`, diff with the manifest);
     - the safety guard (`target.rs`): a pure decision function plus the marked temp folder.
   - Profiles: `projects`, `wide`, `wide-root`, `deep`, `users`, `links`, `names`.
   - `sys::procs::real_home()`.
   - Tests in `crates/ddc-loadgen/tests/`, in the fast gate:
     - determinism (same seed gives the same hash, a different seed a different one);
     - the guard's decision table;
     - each profile's S tree matches its manifest.
     
     A nextest test group caps them at 4 threads.
2. **The rest of the profiles and the golden check.**
   - Profiles `deep-path-max`, `hostile` (a reader hook plus the mutator), `system-root` and `mixed`.
   - `bench/golden/` and `crates/ddc-cli/tests/golden_cli.rs` (ignored tests); `make golden-record` and `make golden-check`, the latter in `make gate`.
   - The seeded-change test.
3. **Disk image and CI load.**
   - `scripts/load-image.sh` and `make load-image PROFILE=… SIZE=M|L|XL`:
     - `hdiutil` with `-owners on`;
     - `.metadata_never_index` and `.fseventsd/no_log`;
     - a detach on exit.
   - A `load` job in `gate.yml` (M) and an L `mixed` step in `nightly.yml`. L and XL times are recorded here.

## Acceptance Criteria

- [ ] `ddc-loadgen` builds every profile at S and M; the same seed and the same `fs_caps` give the same manifest hash.
- [ ] A scan of each profile's S tree except `hostile` finds exactly the manifest's expected findings and totals, and `clean --dry-run` plans exactly its expected items (asserted in the fast gate). For `hostile`, the scan completes without a panic and its findings are a subset of the manifest's static set.
- [ ] `make golden-check` passes on an unchanged tree and fails on a seeded change to one rule (test).
- [ ] The generator refuses a folder on the home's or the boot volume's filesystem without its marker, and a mount point whose `statfs` device does not match the attached image (tested).
- [ ] An L `mixed` tree (10⁶ entries) is generated, scanned and removed on the reference machine; times are recorded in this story.
- [ ] Under `hostile`, a scan completes with no panic. In the deterministic reader test, vanished entries are counted as vanished and their finding is `incomplete`, not an error.

## Final Summary

## Technical Notes

- The generator needs `mkdirat`, `openat`, `linkat` and `utimensat` to build paths past `PATH_MAX`. It uses `rustix` (already in `Cargo.lock` through `tempfile`), because the workspace denies `unsafe` outside `ddc-core/src/sys`. The real home for the guard comes from a new `sys::procs::real_home()` (`getpwuid_r`).
- The `wide-root` profile also exercises a step `aa64c83` left in place: `dir_has_any` for the `requirements*.txt` glob reads the project root once per venv that has no lock file beside it. Story "Complexity guard: operation counts and scaling exponents" asserts it.

- `hdiutil create`, `attach` and `detach` need no `sudo` and raise no permission prompt. Nothing here needs elevated rights.
- An APFS sparse image grows with the data written, and empty files cost metadata only. Expect about 1 KiB per entry: about 10 GB for XL. That is why XL is manual.
- Anything moved to the Trash on the image goes to the image's own `.Trashes`, not the user's Trash. It disappears with the detach.

## Dependencies

- Story "Mandatory gate before every commit".

## Attachments

-

---

*Last updated: 2026-10-28*

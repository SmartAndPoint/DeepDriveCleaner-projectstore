---
type: story
id: "story-synthetic-load-trees-large-wide-deep-and-hostile"
epic: "DDC-005"
title: "Synthetic load trees: large, wide, deep and hostile"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [rust, testing, performance, load]
code_refs: [".config/nextest.toml", ".github/workflows/gate.yml", ".github/workflows/nightly.yml", "Cargo.toml", "Makefile", "bench/golden/**", "crates/ddc-cli/Cargo.toml", "crates/ddc-cli/tests/golden_cli.rs", "crates/ddc-core/src/platform/macos/procs.rs", "crates/ddc-loadgen/**", "scripts/load-image.sh"]
specs: []
started_at: 2026-10-28
closed_at: "2026-10-28T20:28:41.935Z"
plan_updated_at: 2026-10-28
---

# Synthetic load trees: large, wide, deep and hostile

| Field | Value |
|---|---|
| **Epic** | [DDC-005](../epic.md) |
| **Status** | done |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

Load tests need large trees that are the same on every run and every machine. Real homes differ and change. The `aa64c83` quadratic steps showed up only on large system trees with many sibling folders. A tree of a few hundred files hides any O(n²) step.

This story adds a generator. It builds a tree from a profile, a size and a seed, always with the same shape. It writes empty files and files of a few 4 KiB blocks, so a tree costs inodes and little data rather than gigabytes. M and larger trees build on a disposable disk image, so millions of entries never touch the user's volume as entries and cleanup is one detach. S trees (10⁴ entries) go to a marked temporary folder, so the fast gate needs no image.

Profiles (each is a shape real disks have):

| Profile | Shape |
|---|---|
| `projects` | N projects with `.git`, `node_modules` (deep, many small files), `target`, `.venv`, lock files, keep markers |
| `wide` | two flat folders of N/2 entries each, one outside any finding and one a `node_modules` (5·10³ at S, 5·10⁴ at M, 5·10⁵ at L) |
| `wide-root` | one project root with 10⁴+ siblings and many findings inside it, plus requirement files (`requirements.txt`, `package.json`): the shape of `aa64c83` step 2, where every finding re-read the same root. Venvs sit directly under the root (parent = project root) with no lock file beside them, and `requirements.txt` is in the root, so `dir_has_any` reads the root once per venv |
| `deep` | chains nested to a depth of 200 with 1–3-byte names (about N/400 of them, half inside a `node_modules`), and a separate branch of at most 3 levels with 255-byte names (`NAME_MAX`). Both stay under `PATH_MAX` with the fixed root length below (the depth-200 branch tops out near 160 + 200·4 = 960 bytes, the long-name branch near 160 + 3·256 = 928), so `deep` reports no errors: depth 200 with long names would cross `PATH_MAX` within about 4 levels |
| `deep-path-max` | a chain of directories inside a finding (a `node_modules`) up to the `PATH_MAX` boundary, separate from `deep` so a failure names the cause. `PATH_MAX` (1024 on macOS) counts the NUL: the deepest readable directory has a 1023-byte path, and its child, at 1024 bytes or more, fails to open. The generator creates the chain with `mkdirat` relative to a parent handle (absolute paths fail with `ENAMETOOLONG`). The manifest asserts today's behaviour: that child is the one error (`ENAMETOOLONG`, counted in `errors_total`), it makes the finding `incomplete`, and nothing below it is listed. The scan does not abort, and the files listed in the last readable directory still count. A dedicated skip reason would be a schema change, outside this story |
| `users` | per-user folders (hundreds at S, thousands at M), each with caches and small files, the shape of `/Users`, `/private/var/folders` and system trees. One of them, `<tree>/Users/load`, is the home: a fixture for every home-anchored rule, app-owned project artifacts, and a `.git` in the home itself |
| `links` | many hard links to the same inodes, symlink loops, dangling symlinks, symlinks pointing out of the tree |
| `names` | Unicode NFC/NFD pairs, case pairs, spaces, newlines and control characters, names that look like flags. At build time the generator probes the filesystem: normalization- and case-insensitivity, and whether it accepts invalid UTF-8 (APFS refuses it with `EILSEQ`). It records the results in the manifest's `fs_caps` and expects only the names that exist. Invalid UTF-8 is built only where the filesystem accepts it (Linux, DDC-007) |
| `hostile` | unreadable folders, plus a mutator thread that deletes, renames and creates entries while a scan runs |
| `system-root` | a fake data volume root (`Users`, `Library`, `private/var/folders`, `Applications`) for `ddc disk --system-root` |
| `mixed` | every profile but `system-root` (its own root layout) and `hostile` (it changes while scanned), weighted like the reference machine's home |

Size tiers: S = 10⁴ entries (fast gate), M = 10⁵ (CI), L = 10⁶ (nightly and manual), XL = 10⁷ (manual).

## Decomposition

- [x] A workspace tool crate `ddc-loadgen` (`publish = false`, not shipped), with a CLI: `ddc-loadgen build <profile> --entries N --seed S --into <dir> [--device D] [--verify]`, plus `ddc-loadgen manifest <profile>` (APFS `fs_caps` assumed) and `ddc-loadgen mktemp`. It prints a manifest:
  - entry counts;
  - expected findings per rule and expected totals;
  - expected plan items for `clean --dry-run` with the default settings.
- [x] Each profile is its own module. The manifest is computed from the same description that builds the tree, not from running `ddc`.
  - Per-rule fixtures are derived from the catalog's public declarations: `RULES`, each rule's `Matcher`, class, lock files and removal. It follows `rule_fixture` in `ddc-core/tests/common/fixtures.rs`, but stays a copy: `ddc-core`'s tests cannot depend on the generator, which depends on `ddc-core`. A new rule without a fixture fails the generator's tests, as it fails the existing ones.
  - The expected class and selection repeat only the tree-dependent rules: app-owned and no-lockfile Review (`rules/classify.rs`), keep markers, and project activity against `min_age`. They never call `scan` or `classify`.
  - Every `Matcher` kind is placed by one exhaustive `match` (project, home, synthetic or unlisted), with no catch-all arm. A new kind fails to compile, and a listed rule of a kind no profile builds panics in a test.
  - Keep markers are planted by the `projects` profile, not derived from the catalog.
  - Each branch of the model has a fixture: a monorepo (lock file in the git root, one package edited today, one idle, a venv whose `requirements-dev.txt` sits in the package only), a CMake build folder with no project around it (unknown activity, not selected), app-owned artifacts under a dot folder and under `Library` in the home (Review), and a `.git` in the home that does not make the home a project root.
- [x] Sizes are the same on every filesystem. Payload files are exactly 4096·k bytes long, with non-zero content. Marker files whose content a matcher reads (`CACHEDIR.TAG`, `CMakeCache.txt`, `pyvenv.cfg`, toolchain pins) are padded to exactly 4096 bytes with trailing newlines (every parser of these files skips blank lines or trims). Everything else is empty. Symlink targets stay under 60 bytes, which ext4 stores without a block (DDC-007). The tree root, after `canonicalize()`, is padded to a fixed length of 160 bytes, so path lengths, the `PATH_MAX` boundary and the order of findings are the same on every machine.
- [x] Home. Most catalog rules are anchored at the home folder (`Catalog::builtin(home)`). Each profile's description names the home its manifest assumes: none (content-matched project rules only) or a fake home inside the tree (`<tree>/Users/load` for `users` and `mixed`, `<tree>/System/Volumes/Data/Users/load` for `system-root`). `verify` passes it to the scan through `Env::with_paths`; a profile without a home gets an empty folder beside the tree. Home-anchored findings are expected only when the profile provides a home. It is never the real home.
- [x] Modification times are backdated by the description (older than the default `min_age` of 14 days where a finding is expected, newer where it is not), so the expected plan is exact.
- [x] Safety guard, by filesystem identity, not by path prefix. Two target kinds are allowed:
  - M and above on macOS: a volume the generator mounted itself. Before the first write it calls `statfs` on the target and checks that `f_mntfromname` equals the device `hdiutil attach -plist` returned, and that the volume's root is named `ddc-load-<nonce>`, is the user's, and holds a `.ddc-loadgen` marker with that nonce (written by `load-image.sh` right after the attach).
  - S trees (all platforms) and everything on Linux and Windows: a fresh directory under the system temporary folder (`$TMPDIR`), holding a marker file the generator created. The marker check is by identity, not by name alone: a direct child of `$TMPDIR` named `ddc-load-<nonce>`, owned by the user, whose `.ddc-loadgen` holds that nonce (so a copied marker or a nested `ddc-load-*` folder is refused).
  
  In both cases it refuses any target on the same filesystem as the real home or `/` (compared by `st_dev`, with the real home from `getpwuid(getuid())`, not from `$HOME`), unless it is the marked temporary folder. So the fast gate needs no disk image. When the password database names no home, the guard fails closed.

  On macOS a marked temporary folder takes S trees only (`TEMP_MAX` = 10⁴ entries); a larger one is refused with a pointer to `make load-image`. On Linux and Windows, with no image, the folder takes any size. The marker proves the volume was marked by a `load-image.sh` run, not that it is a disk image: an external disk prepared the same way by hand would pass. Against accidents that is enough; a check that `--device` is listed by `hdiutil info` would close it.
- [x] macOS `make load-image LOAD_PROFILE=<profile|all> LOAD_SIZE=S|M|L|XL` (`scripts/load-image.sh`):
  - creates an APFS sparse bundle (`hdiutil create -type SPARSEBUNDLE`) in the user's temporary folder, named `ddc-image-<pid>-<nonce>`. At S it built `projects` in 4.5 s, against 54 s for a single-file sparse image and 84 s for a UDIF image;
  - first discards an image left by an interrupted run, found by the process in its name being gone;
  - attaches it with `hdiutil attach -nobrowse -owners on -plist` at `/Volumes/ddc-load-<nonce>` (no `sudo` needed); `create` and `attach` are retried, as hosted runners now and then answer "Resource busy";
  - writes `.metadata_never_index`, `.fseventsd/no_log` and the guard's marker at the root, so Spotlight and fseventsd leave millions of entries alone (`mdutil -s` reports indexing disabled);
  - builds every asked profile in its own folder with `ddc-loadgen build --verify --brief`, prints build and scan times per profile, and fails if any profile differs from its manifest;
  - detaches the image by its whole disk and deletes it afterwards; when the detach fails it keeps the image for the next run's sweep.
- [x] Generation time per tier is recorded (Implementation Plan, step 3). L must build in ≤ 5 min on the reference machine; `mixed` at L built in 3 min 52 s, so no tier is cached as an image.
- [x] A test in the fast gate: S-size trees of every profile generate identically for the same seed and the same `fs_caps` (the manifest hash is stable). Only `names`, and `mixed` through its `names` part, depend on `fs_caps`.
- [x] Golden check. `make golden-record` and `make golden-check`: for every S profile except `hostile`, run `scan --json` and `disk --json` (on `system-root`) with a fixed `HOME` and `--data-dir`, and `rules --json` once (the catalog does not depend on the tree). Normalise what differs between runs and machines, store the outputs under `bench/golden/`, and diff:
  - times: `generated_at`, `duration_ms`, `as_of`, idle days, and a `last_activity` other than the fixed old date (it becomes `<build>`; the old date stays verbatim);
  - the run's folder (replaced by `<run>`, since `$TMPDIR` differs per user and machine);
  - inode numbers, and device numbers as `<tree-dev>` or `<other-dev>`, so crossing a device still shows;
  - the build: `ddc_version`; the machine: `volumes`, `container`, `full_disk_access`, and in `not_counted` what the Data volume holds besides the tree (`bytes`, `data_volume_bytes`; `counted_bytes` and `holds` stay).

  It preserves walk and rule behaviour: the CLI runs with an empty `PATH`, so tool-backed areas of `disk` (snapshots, simulators, Homebrew) are not exercised, and the plan is checked through each finding's `selected` and removal here and through `verify`'s manifest comparison, not a golden `clean --dry-run`. The goldens of `names` and of `mixed` (which holds a `names` part) are keyed by their `fs_caps` (`scan-cini.json` on APFS); they exist for APFS only, so DDC-007 records its file systems' variants before its gate can pass. This is the behaviour-preservation check DDC-006's refactors and DDC-007's move and catalog split use.
- [x] `hostile`: the mutator deletes, renames and creates filler inside a dedicated `churn/` folder while a scan runs, and the scan finishes. The filler matches no rule. A folder replaced under the same name (renamed away and re-created) shows up as `Changed` (counted in `skipped_total`), not as vanished; one renamed away and not replaced is vanished. An entry that vanishes between listing and reading is counted in the walk's `vanished` and makes its owning finding `incomplete`. It is not an error and does not abort the scan. A deterministic test wraps the directory reader (`scan::walk_with_reader`) to delete an entry just before it is read, to report one as gone between listing and metadata, and to replace a folder under the same name, and asserts the `vanished` and `skipped_total` counts. Locked folders are counted once as a folder and an error, their contents unseen, and make their finding `incomplete`; the manifest models them (`Desc::lock`). Running as root would read them anyway: the manifest assumes an unprivileged user (CI runs as `runner`; root containers are DDC-007's).

## Implementation Plan

Planned with `projectstore:planner` on 2026-10-28. Three commits; the first is useful on its own.

1. **Generator and S manifests.**
   - The crate `crates/ddc-loadgen` (lib + bin):
     - a SplitMix64 RNG;
     - the description type, and the build from it over parent file descriptors, with hard links and then mtimes set in later passes;
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

   Done in `919f896`. Measured on the reference machine (release build, seed 1, in a marked `$TMPDIR` folder on the Data volume: the image lands in step 3): every profile at M (10⁵ entries) verifies with no diff. Build plus verify per profile at M: `projects` 12.5 s, `wide` 12.1 s, `wide-root` 10.7 s to build and 26 s in all (the `dir_has_any` re-read of the root shows here, see Technical Notes), `deep` 18.4 s, `users` 13.2 s, `links` 33 s, `names` 12.9 s. File data at M, from the manifests' `scanned_bytes` (metadata not included): `projects` 217 MiB (about 2.3 KB per entry), `wide` 147 MiB, `users` 144 MiB, `deep` and `names` 96 MiB, `links` 51 MiB, `wide-root` 16 MiB. At S, `wide` holds 5·10³ entries per folder and `users` about 420 user folders.
2. **The rest of the profiles and the golden check.**
   - Profiles `deep-path-max`, `hostile` (a reader hook plus the mutator), `system-root` and `mixed`.
     - The builder keeps one open handle per level (`build::walk`), about 430 at the `deep-path-max` boundary, past the default soft limit of 256: raise it through `sys::rlimit` or reopen from a shallower handle. `deep` at 200 levels is already close.
     - `verify` gains a subset mode for `hostile`: findings must be a subset of the manifest's static set (AC 2); today every unexpected finding is a difference.
   - `bench/golden/` and `crates/ddc-cli/tests/golden_cli.rs` (ignored tests); `make golden-record` and `make golden-check`, the latter in `make gate`.
     - A `ddc-loadgen clean` subcommand removes the generator's own `ddc-load-*` folders, by the guard's identity rule (only those holding its marker); the golden targets call it afterwards, so the gate leaves nothing in `$TMPDIR`.
   - A pinned S hash per profile in `tests/determinism.rs`, so an unintended change to the generator fails before the golden check does.
   - The seeded-change test.

   Done in `54336bb`, reviewed with `projectstore:reviewer`. Changes from the plan:
   - The golden tests are ordinary tests, not ignored: they run in `nextest`, so in `gate-fast` before every commit as well as in `make gate`, in the load test group. `make golden-check` runs only them. The load group (generator tests and golden) took about 50 s of wall time on 4 threads on the reference machine.
   - Test trees are removed on drop (`target::Scratch`, which first gives back the permissions `hostile` took), not by `clean` afterwards: `clean` cannot tell an abandoned tree from one a run in another worktree is using. `ddc-loadgen clean` and `make loadgen-clean` stay for trees left by manual `mktemp` runs and killed processes. Nothing in the gate runs it; what a killed run leaves stays until someone does. No test yet asserts that a dropped `hostile` or `deep-path-max` tree is gone (step 3).
   - `mixed` weights `projects` 35, `users` 20, `wide` 10, `wide-root` 8, `deep` 10, `deep-path-max` 5, `links` 7, `names` 5. It leaves out `system-root` (its own root layout) and `hostile` (it changes while scanned), so its golden stays exact.
   - `deep-path-max` builds chains of `node_modules/x/x…` in a project, each level with a file; at S the manifest expects 12 errors, one per chain, and every finding `incomplete`. The builder raises the open-file limit through `sys::rlimit`.
   - The seeded change switches `python.venv` off in `settings.json`, which changes the findings the way a changed rule does.
   - The golden set is 2.4 MB; `names` alone is about 1.3 MB (1656 findings).
3. **Disk image and CI load.**
   - `scripts/load-image.sh` and `make load-image PROFILE=… SIZE=M|L|XL`:
     - `hdiutil` with `-owners on`;
     - `.metadata_never_index` and `.fseventsd/no_log`;
     - a detach on exit.
   - A `load` job in `gate.yml` (M) and an L `mixed` step in `nightly.yml`. L and XL times are recorded here.
   - Left from step 1's review:
     - the image check above, and the tier-to-target decision.
   - Left from step 2: a test that a dropped `hostile` or `deep-path-max` tree is gone; M builds of `deep-path-max`, `system-root`, `hostile` and `mixed` (AC 1 then had M for the first seven only; it now covers all eleven).

   Done in `6983243`, reviewed with `projectstore:reviewer`. Measured on the reference machine (release build, seed 1, on the image) while other sessions kept it busy (load average 25–64), so the times are upper bounds. Every profile at M verified with no difference:

   | Profile | Entries | Build (s) | Scan (s) |
   |---|---|---|---|
   | `projects` | 100 027 | 11.0 | 0.9 |
   | `wide` | 100 001 | 14.6 | 0.1 |
   | `wide-root` | 100 000 | 17.2 | 18.2 |
   | `deep` | 100 242 | 19.0 | 4.6 |
   | `deep-path-max` | 100 556 | 22.7 | 10.4 |
   | `users` | 100 006 | 12.0 | 2.0 |
   | `links` | 100 004 | 61.6 | 0.1 |
   | `names` | 100 101 | 17.3 | 2.9 |
   | `system-root` | 100 013 | 14.1 | 1.4 |
   | `hostile` | 100 004 | 14.2 | 1.0 |
   | `mixed` | 100 041 | 19.7 | 1.9 |

   On GitHub's `macos-26` runner (Gate run 40762150384) the same eleven verified with no difference; builds took 13–78 s per profile (`links` the slowest, `wide-root`'s scan 19.6 s), and the `load` job 8 minutes in all.

   All eleven at M take 1.8 GiB on the volume and 1.9 GB of image. `links` took 346 s in the run of all eleven, while another process saturated the Data volume's I/O, and 61.6 s alone; hard links are its cost. `wide-root`'s 18 s scan is the `dir_has_any` re-read of the root (Technical Notes).

   `mixed` at L: 1 000 123 entries (289 164 folders, 710 834 files, 1.5 GB of file data), built in 231.9 s, scanned and checked in 51.6 s with no difference, 5 min 4 s with the image's create, attach, detach and delete. The image took 1.9 GB and was gone afterwards. XL is not run: it is manual, at about ten times L.

   Changes from the plan:
   - A sparse bundle, not a single-file sparse image, from the measurement above.
   - The image check is the marker at the volume root, not `hdiutil info` (Safety guard).
   - The Gate workflow's new `load` job builds every profile at M; `make dist` now waits for it too, since `scripts/ci-green.sh` reads the whole run. The nightly workflow builds `mixed` at L in a job of its own, so a gate or coverage failure does not skip it. Each uploads its per-profile JSON (`target/load`).
   - The Makefile variables are `LOAD_PROFILE`, `LOAD_SIZE` and `LOAD_SEED`, so a generic `PROFILE` in the environment does not leak in.
   - Every profile test and the mutator test now assert that the dropped tree is gone (left from step 2).

## Acceptance Criteria

- [x] `ddc-loadgen` builds every profile at S and M; the same seed and the same `fs_caps` give the same manifest hash — evidence: `crates/ddc-loadgen/tests/determinism.rs` (pinned S hashes), `make load-image LOAD_PROFILE=all LOAD_SIZE=M` locally and in the Gate `load` job (run 40762150384), 0 differences; the M hashes in that run's `load-m` artifact equal the local ones for all eleven profiles.
- [x] A scan of each profile's S tree (`hostile` while static) finds exactly the manifest's expected findings and totals, and `clean --dry-run` plans exactly its expected items (asserted in the fast gate). Under the mutator, `hostile`'s scan completes without a panic and its findings are a subset of the manifest's static set — evidence: `crates/ddc-loadgen/tests/profiles.rs`, `crates/ddc-loadgen/tests/hostile.rs::a_scan_completes_while_the_tree_changes`.
- [x] `make golden-check` passes on an unchanged tree and fails on a seeded change to one rule (test) — evidence: `crates/ddc-cli/tests/golden_cli.rs::a_change_to_one_rule_fails_the_golden_check`, `make golden-check`.
- [x] The generator refuses a folder on the home's or the boot volume's filesystem without its marker, and a mount point whose `statfs` device does not match the attached image (tested). Tested in `tests/guard.rs`, the unmarked image and the too-large temporary tree included — evidence: `crates/ddc-loadgen/tests/guard.rs::decision_table`, `::the_real_home_and_the_boot_volume_are_refused`, `::a_mount_that_is_not_the_attached_image_is_refused`.
- [x] An L `mixed` tree (10⁶ entries) is generated, scanned and removed on the reference machine; times are recorded in this story — evidence: `make load-image LOAD_PROFILE=mixed LOAD_SIZE=L` on 2026-10-29 local (2026-10-28 UTC), 1 000 123 entries, built in 231.9 s, scanned in 51.6 s, 0 differences, image removed (Implementation Plan, step 3); brief output in Attachments.
- [x] Under `hostile`, a scan completes with no panic. In the deterministic reader test, vanished entries are counted as vanished and their finding is `incomplete`, not an error — evidence: `crates/ddc-loadgen/tests/hostile.rs::a_vanished_or_replaced_entry_makes_its_finding_incomplete`.

## Final Summary

**What changed.** A workspace tool crate, `ddc-loadgen`, builds deterministic load trees from a profile, a size and a seed, and derives from the same description a manifest of what a scan must report. There are eleven profiles: `projects`, `wide`, `wide-root`, `deep`, `deep-path-max`, `users`, `links`, `names`, `system-root`, `hostile` and `mixed`. Three commits:
- `919f896`: the generator, seven profiles, the guard;
- `54336bb`: the remaining profiles, the golden check, pinned hashes;
- `6983243`: the disk image and the CI jobs.

**How each tier runs:**
- **S trees** build in a marked temporary folder inside every gate. There, the S scan is checked exactly against its manifest, and `scan`, `disk` and `rules` JSON are compared with `bench/golden/`.
- **M and larger trees** build on a disposable APFS sparse bundle (`make load-image`). CI builds all eleven profiles at M on every push. The nightly run builds `mixed` at L.

**Why.** The quadratic steps fixed in `aa64c83` showed only on large real trees. These trees make scale, hostile file systems and behaviour changes testable on every run and every machine, and give DDC-006's refactors and DDC-007's move a behaviour-preservation check.

**Tests executed:**
- `make gate`: 431 tests;
- Gate runs 40738831087, 40746829361 and 40762150384, all green;
- `make load-image` locally:
  - all profiles at S and at M;
  - `mixed` at L;
  - each with 0 differences, and each image removed afterwards.

**Risks and follow-ups:**
- The golden files of `names` and `mixed` exist for case-insensitive, normalization-insensitive APFS only (`scan-cini.json`). A case-sensitive APFS Data volume fails `golden_names` and `golden_mixed` in `gate-fast`, with a pointer to `make golden-record`, which would add a second variant. DDC-007 records the variants for its file systems.
- The image check is a marker, not `hdiutil info`. See Safety guard.
- The manifest assumes an unprivileged user. Root reads locked folders.
- `make dist` now waits for the M `load` job (about 8 minutes on the runner).
- `wide-root` shows the `dir_has_any` re-read of the root: an 18 s scan at M. Story "Complexity guard: operation counts and scaling exponents" asserts and removes it.
- XL (10⁷ entries, about 20 GB of image) is manual and has not been run.
- The nightly `load` job (`mixed` at L) has not yet run on a hosted runner: the only Nightly run (40723880175) predates it. Its first scheduled run is 2026-10-29 03:00 UTC; until it is green, L on the runner is untested (M built 2–3 times slower there than locally).
- The load test group (S trees of eleven profiles and the golden check) runs in `gate-fast` before every commit, about 50 s of wall time on 4 threads.

## Technical Notes

- The generator needs `mkdirat`, `openat`, `linkat` and `utimensat` to build paths past `PATH_MAX`. It uses `rustix` (already in `Cargo.lock` through `tempfile`), because the workspace denies `unsafe` outside `ddc-core/src/sys`. The real home for the guard comes from a new `sys::procs::real_home()` (`getpwuid_r`).
- The `wide-root` profile also exercises a step `aa64c83` left in place: `dir_has_any` for the `requirements*.txt` glob reads the project root once per venv that has no lock file beside it. Story "Complexity guard: operation counts and scaling exponents" asserts it.

- `hdiutil create`, `attach` and `detach` need no `sudo` and raise no permission prompt. Nothing here needs elevated rights.
- An APFS sparse bundle grows with the data written, and empty files cost metadata only. File data runs 0.2–2.3 KB per entry by profile (step 1's M figures), plus metadata. Measured: `mixed` at L took 1.9 GB of image for 1.5 GB of file data; XL would be about 20 GB. That is why XL is manual.
- Anything moved to the Trash on the image goes to the image's own `.Trashes`, not the user's Trash. It disappears with the detach.

## Dependencies

- Story "Mandatory gate before every commit".

## Attachments

- `L-mixed.json`, the brief output of `make load-image LOAD_PROFILE=mixed LOAD_SIZE=L` (2026-10-29 local): entries 1 000 123 (289 164 folders, 710 834 files, 59 errors, 1 502 846 976 bytes scanned), hash `d4c71c4a605ae30e`, `build_ms` 231867, `verify_ms` 51607, `difference_count` 0, on `/Volumes/ddc-load-6b31b16f` (gone afterwards).

---

*Last updated: 2026-10-28*

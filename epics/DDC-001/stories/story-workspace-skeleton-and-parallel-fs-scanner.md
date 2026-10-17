---
type: story
id: "story-workspace-skeleton-and-parallel-fs-scanner"
epic: "DDC-001"
title: "Workspace skeleton and parallel filesystem scanner"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-17
updated: 2026-10-17
external_refs: {}
tags: [rust, scanner]
code_refs: ["Cargo.toml", "rust-toolchain.toml", "clippy.toml", "rustfmt.toml", "Makefile", "crates/ddc-core/src/lib.rs", "crates/ddc-core/src/env.rs", "crates/ddc-core/src/sys/", "crates/ddc-core/src/scan/", "crates/ddc-core/tests/scanner.rs", "crates/ddc-cli/src/main.rs", "crates/ddc-cli/src/cmd/du.rs", "crates/ddc-cli/src/cmd/diag.rs", "crates/ddc-cli/tests/du_cli.rs", "crates/ddc-cli/tests/diag_cli.rs", "bench/"]
specs: []
started_at: "2026-10-17T09:24:26.866Z"
closed_at: "2026-10-17T10:03:12.674Z"
plan_updated_at: "2026-10-17T09:24:26.866Z"
---

# Workspace skeleton and parallel filesystem scanner

| Field | Value |
|---|---|
| **Epic** | [DDC-001](../epic.md) |
| **Status** | done |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The foundation of all of `ddc`: a Cargo workspace per [[adr/rust-core-library-and-cli-with-json-output|Rust core: library + CLI with JSON output]] and a parallel filesystem walk in `ddc-core`. All other stories depend on the scanner. It must be fast (`getattrlistbulk`), count **allocated** and **private** (freeable) space, and give rules a hook on every directory, so that a found artifact (for example, `node_modules`) is counted as one unit, not as hundreds of thousands of files.

## Decomposition

- [x] Workspace at the repository root: `rust-toolchain.toml` (1.99.1, rustfmt, clippy), `Cargo.toml` (resolver 3, edition 2024, `rust-version = 1.85` (later raised to 1.89), `workspace.lints`), `clippy.toml`, `Makefile` (absolute `CARGO`, export `PATH=$HOME/.cargo/bin:$PATH`), `.gitignore`, `git init`; the empty `deepdrivecleaner/` is removed
- [x] `bench/baseline-2026-10-17.tsv` and `bench/make_baseline.py` from the research data (rule ids as in the catalog)
- [x] `Env`: home directory, data directory (`--data-dir`/`DDC_DATA_DIR`), log directory, current time; the only place where `$HOME` is read (the `--data-dir` CLI flag arrives with its first consumer, `ddc scan`)
- [x] `sys` module (the only one with `unsafe`): `getattrlistbulk` (name, type, `fileid`, `devid`, `flags`, `linkcount`, `allocsize`, entry error; `ATTR_CMNEXT_PRIVATESIZE` on request), `setiopolicy_np`, raising `RLIMIT_NOFILE`, `statfs`
- [x] `DirReader` trait: `BulkReader` (default) and `StdReader` (`read_dir` + `symlink_metadata`) for cross-check tests and non-APFS volumes
- [x] Walker on `rayon::scope`: one task per directory, the descriptor is closed before child tasks are spawned; opening by path with `O_DIRECTORY|O_NOFOLLOW` and checking `fstat` against the `(dev, ino)` from the parent's listing; a different `st_dev` is not walked; `EDEADLK` and `SF_DATALESS` are "skipped (dataless)"
- [x] Two-phase directory hook: before reading (exclusions, including `faccessat(<dir>/.ddc-keep)`, known paths, "do not walk") and after reading (rules by name and contents); decisions `Descend` / `Claim(rule)` / `Region(rule)` / `Skip`; inside a `Claim`, `ATTR_CMNEXT_PRIVATESIZE` is requested
- [x] Hard links (`nlink > 1`): a `(dev, ino) → owners` table, resolved after the walk: all links inside one finding → its `bytes`, otherwise → the `shared_bytes` of each; in the total an inode is counted once
- [x] Access errors are collected (path + kind) up to a limit and counted, and they do not abort the scan; `NotFound` is "disappeared during the scan"
- [x] Atomic progress counters (files, directories, bytes, errors) and a cancel flag
- [x] Hidden command `ddc du <path>` for cross-checking against `du` and for measurements

## Implementation Plan

Route based on the `projectstore:planner` plan (2026-10-17) and the `getattrlistbulk` prototype. There is no spec; the contracts are the epic's two ADRs.

1. **Repository housekeeping.** Remove the empty `deepdrivecleaner/`, `git init`, `.gitignore` (`/target/`, `.DS_Store`, `.claude/settings.local.json`).
2. **Workspace.** `rust-toolchain.toml` (1.99.1, `rustfmt`, `clippy`, profile minimal); root `Cargo.toml` (members `crates/*`, resolver 3, `workspace.package` edition 2024 / `rust-version = 1.85` (later raised to 1.89), `workspace.dependencies`, `workspace.lints`: `unsafe_code = "deny"`); `clippy.toml` (`disallowed-methods`: `std::env::home_dir`, `std::fs::remove_dir_all`/`remove_dir`/`remove_file`); `Makefile` (`CARGO := $(HOME)/.cargo/bin/cargo`, `export PATH := $(HOME)/.cargo/bin:$(PATH)`; targets `fmt`, `clippy`, `test`, `check`, `release`).
3. **`ddc-core`.**
   - `lib.rs` — `#![deny(clippy::print_stdout, clippy::print_stderr, clippy::dbg_macro)]`, re-exports.
   - `env.rs` — `Env { home, data_dir, log_dir, now }`, `Env::from_process()` (the only read of `$HOME`/`DDC_DATA_DIR`) and `Env::for_test(root)`.
   - `sys/` (the only module with `#![allow(unsafe_code)]`): `bulk.rs` (`open(O_DIRECTORY|O_NOFOLLOW|O_CLOEXEC)` + `fstat` + a `getattrlistbulk` loop, parsing entries into `EntryMeta { name, kind, dev, ino, nlink, flags, alloc, private: Option<u64> }`), `iopolicy.rs` (`setiopolicy_np`/`getiopolicy_np` for dataless), `rlimit.rs` (`RLIMIT_NOFILE` up to `min(hard, 10240)`).
   - `scan/reader.rs` — the `DirReader { fn read(&self, dir, want_private) -> Result<DirListing, ReadError> }` trait; `BulkReader`, `StdReader`.
   - `scan/walker.rs` — `rayon::ThreadPool` + `scope`; one task per directory; checking the `(dev, ino)` of the opened directory against the parent's listing; a two-phase `Visitor` (`before_read` → `Pre::{Continue, Claim, Region, Skip}`, `after_read` → `Post::{Descend, Claim}`); owners (`OwnerId`) with atomic `bytes/private/files/dirs/errors` counters; top largest files through `on_file`.
   - `scan/inodes.rs` — a sharded `(dev, ino) → {alloc, nlink, seen, owners}` table; resolution after the walk (exclusive → `bytes`, otherwise `shared_bytes`).
   - `scan/progress.rs` — atomic counters and a cancel flag.
   - `scan/mod.rs` — `walk(roots, visitor, opts, progress) -> WalkOutput` (roots canonicalized, nested ones dropped); `NullVisitor`.
4. **`ddc-cli`.** `main.rs` (at startup: dataless OFF and `RLIMIT_NOFILE`), `cli.rs` (`clap`, hidden `du`), `cmd/du.rs` (`<KiB>\t<path>` to stdout, statistics and time to stderr; `--reader bulk|std`, `--threads`).
5. **Tests.** `crates/ddc-core/tests/common/mod.rs` — `TreeBuilder` (canonicalized tempdir, a "not inside the real `$HOME`" check, `dir/file/sparse/symlink/hardlink/clone/chmod/mtime`, `snapshot()`); `tests/scanner.rs` — criteria 1–6; `crates/ddc-cli/tests/du_cli.rs` — `ddc du <tmp>` against `/usr/bin/du -skx <tmp>`.
6. **Measurements.** Release build; `ddc du ~/Projects` — warm-up + 3 runs, median; cross-check against `du -skx` (criteria 7–8); the result goes into the final summary.
7. **`make check`** (criterion 9).

Files touched: `Cargo.toml`, `Cargo.lock`, `rust-toolchain.toml`, `clippy.toml`, `Makefile`, `.gitignore`, `bench/`, `crates/ddc-core/{Cargo.toml,src/lib.rs,src/env.rs,src/sys/,src/scan/,tests/common/,tests/scanner.rs}`, `crates/ddc-cli/{Cargo.toml,src/main.rs,src/cli.rs,src/cmd/du.rs,tests/du_cli.rs}`.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [x] Tests on temporary trees: allocated size matches `lstat` `blocks*512` (a sparse file is 0); a symlink to a directory and a symlink loop are not dereferenced; an access error (`chmod 000`) is collected and the siblings are counted; a `Claim` subtree is attributed to the finding, and a nested match does not produce a separate finding — evidence: `crates/ddc-core/tests/scanner.rs` `allocated_size_matches_lstat_blocks`, `symlinks_are_not_followed`, `unreadable_directory_is_reported_and_siblings_are_counted`, `claimed_subtree_is_attributed_to_its_owner` (`make check`, 16/16)
- [x] Hard-link tests: both links inside one finding → `bytes`; links in two findings → `shared_bytes` for both; the grand total counts the inode once — evidence: `scanner.rs` `hard_links_inside_one_finding_are_exclusive`, `hard_links_across_findings_are_shared_by_both`, `hard_link_to_a_file_outside_the_scan_is_shared`
- [x] Test: a `cp -c` clone of a large file has allocated size = the original's size and private size ≈ 0; a finding's private size does not include inodes that went to `shared_bytes` — evidence: `scanner.rs` `apfs_clone_has_near_zero_private_size`, `hard_links_across_findings_are_shared_by_both` (private ≤ exclusive bytes, the second finding has `Some(0)`)
- [x] Determinism test: two walks of the same tree give the same result (except for time) — evidence: `scanner.rs` `walk_is_deterministic` (6 walks, cross-finding hard links, 4 threads)
- [x] Test: `BulkReader` and `StdReader` give the same totals on the same tree; `ddc du <tmp>` matches `/usr/bin/du -skx <tmp>` — evidence: `scanner.rs` `bulk_and_std_readers_agree`, `bulk_reader_matches_lstat_per_entry_across_batches` (3,000+ entries, several batches, 255-byte names / UTF-8 / NFD / `\t` / `\n`, resource fork, FIFO, hard link); `crates/ddc-cli/tests/du_cli.rs` `ddc_du_matches_system_du`
- [x] Test: after startup, `getiopolicy_np(IOPOL_TYPE_VFS_MATERIALIZE_DATALESS_FILES, IOPOL_SCOPE_PROCESS)` returns `IOPOL_MATERIALIZE_DATALESS_FILES_OFF` — evidence: `crates/ddc-cli/tests/diag_cli.rs` `ddc_starts_with_dataless_materialization_off_and_raised_file_limit` (runs the binary, `ddc diag` → `dataless_materialization: off`)
- [x] The `ddc du ~/Projects` total matches `du -skx ~/Projects` within 1% — evidence: 2026-10-17, `ddc du ~/Projects` = 126,443,472 KiB, `/usr/bin/du -skx ~/Projects` = 126,443,472 KiB (0.000%); earlier reproduced independently by `projectstore:reviewer` (126,392,864 = 126,392,864)
- [x] `ddc du ~/Projects` ≤ 10 s (median of three runs after a warm-up; the prototype showed 6.8–8.8 s), measurement in the final summary — evidence: release, warm-up + 3 runs: 8.19 / 7.86 / 7.83 s, median 7.86 s (load average 5–6)
- [x] `make check` (fmt, clippy `-D warnings`, test) passes — evidence: `make check` on the current tree: fmt and clippy clean, 33 tests green (scanner 16, rules 15, du_cli 1, diag_cli 1)

## Final Summary

**What was done.** A Cargo workspace at the repository root (`crates/ddc-core`, `crates/ddc-cli` with the `ddc` binary, later renamed to `deepdrivecleaner` with a `ddc` symlink via `make install`), toolchain 1.99.1 pinned, a `Makefile` that works around the broken Homebrew Rust. Scanner: our own parallel walk on `rayon::scope` on top of `getattrlistbulk` (all `unsafe` is in `ddc-core/src/sys`; buffer parsing is safe code with bounds checks), a fallback `StdReader`, a two-phase `Visitor` hook (`Pre`/`Post`: `Claim`/`Region`/`Skip`), deferred and deterministic hard-link accounting (exclusive → `bytes`, otherwise `shared_bytes`), APFS private size at the visitor's request, `errors`/`skipped` counters on each finding, dataless materialization turned off, a raised `RLIMIT_NOFILE`, `statfs`. Hidden commands `ddc du` and `ddc diag`. `bench/`: the reference machine baseline and its generator.

**Measurements.** `~/Projects` (2.24 million files, 233 thousand directories, 120.6 GiB): `ddc du` — 7.86 s (median), `du -skx` — 41.9 s, the total matches byte for byte. More than 10 threads gives no speedup (20 → 8.5 s, 32 → 8.7 s). Private size for all files triples the time (30 s), so the rule catalog requests it only where clones are expected.

**From the review (`projectstore:reviewer`).** Both blockers are closed (a post-startup state test through `ddc diag`; `make check` on the final tree), and so are the should-fix items: a failed re-read with private size is now recorded as an error, and "not obtained" → `private_bytes: None` instead of substituting the allocated size; a finding has `skipped`; `sys::statfs` was added; `__pycache__/` is in `.gitignore`; a test of entry parsing against `lstat` across several batches. Minor items: `ENOENT` on an entry → "disappeared"; the recorded errors are a deterministic subset (the smallest paths); the undocumented `FSOPT_NOFOLLOW` was removed; an error entry without a name does not bring down the directory; silent test skips print `SKIPPED`; `std::env::var*` are forbidden by clippy outside `Env`.

**Risks and follow-up.** Paths ≥ 1024 bytes are not read (same as `du`; `openat` could handle them). The number of threads is not limited by `RLIMIT_NOFILE` (the soft limit is raised at startup). A `Region` overlapped by a `Claim` of the same directory leaves an empty region. The changes are not committed; the commit happens at the user's request.

## Technical Notes

- `getattrlistbulk` buffer format: for each entry, a `u32` length (8-byte alignment), the `attribute_set_t` of returned attributes, then `ATTR_CMN_ERROR` (if requested), then the attributes in ascending bit order: common → dir → file → fork/cmnext; each attribute is aligned to 4 bytes, read with `read_unaligned`. An `attrreference` offset is relative to the reference itself.
- `ATTR_CMNEXT_*` work only with `FSOPT_ATTR_CMN_EXTENDED` and live in `forkattr`.
- Private size is three times more expensive (22 s vs 7 s on `~/Projects`), so it is requested only inside findings.
- The soft open-files limit under launchd is 256: no `par_iter` while holding an open descriptor; raise `RLIMIT_NOFILE` to `min(hard, 10240)`.
- In edition 2024, external functions are declared as `unsafe extern "C"`; `setiopolicy_np` is not in `libc`, so declare it by hand.
- `~/Library/Containers` and similar directories return `EPERM` without Full Disk Access. This is expected and goes into the report.

## Dependencies

- rustup stable ≥ 1.89 (`File::try_lock` for the data directory lock; edition 2024), raised from 1.85 by the story "ddc scan report"; `cargo` is called only with `PATH="$HOME/.cargo/bin:$PATH"`

## Attachments

-

---

*Last updated: 2026-10-17*

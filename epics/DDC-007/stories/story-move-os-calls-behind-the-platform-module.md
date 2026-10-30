---
type: story
id: "story-move-os-calls-behind-the-platform-module"
epic: "DDC-007"
title: "Move OS calls behind the platform module"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-30
external_refs: {}
tags: [rust, refactor, portability]
code_refs: []
specs: []
started_at: "2026-10-30T10:15:51.106Z"
closed_at: null
plan_updated_at: "2026-10-30T10:15:51.106Z"
---

# Move OS calls behind the platform module

| Field | Value |
|---|---|
| **Epic** | [DDC-007](../epic.md) |
| **Status** | in-progress |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

This story does the move the seam ADR describes. Afterwards the workspace compiles for Linux and Windows, macOS behaves exactly as before, and the build matrix can run tests natively on all three systems. It is p1 because the build matrix's stage 2 waits on it.

OS-specific code today, found by `grep -rn 'libc::\|std::os::unix\|osascript\|launchctl'`:

| Where | What |
|---|---|
| `ddc-core/src/sys/*` | all FFI (bulk listing, fsops, procs, statfs, iopolicy, rlimit, signals, term) |
| `scan/walker.rs`, `scan/reader.rs` | `libc` errno mapping to `ReadErrorKind`, `std::os::unix` metadata |
| `report/backups.rs`, `report/contents.rs` | `libc::O_NOFOLLOW` in `backups.rs`; `libc::EACCES` in `contents.rs`, test module only |
| `report/after_clean.rs`, `rules/matcher.rs` | `std::os::unix` |
| `exec/system.rs`, `exec/delete.rs`, `exec/guard.rs` | `SIGKILL` of process groups, `std::os::unix` |
| `disk/mod.rs`, `disk/visitor.rs`, `disk/apfs.rs`, `disk/*` | macOS areas, APFS |
| `env.rs` | `HOME`, `~/Library` folders |
| `ddc-cli/src/launchd.rs`, `notify.rs`, `cmd/schedule.rs` | launchd, `osascript`, `std::os::unix` |
| `ddc-cli/src/main.rs`, `progress.rs`, `cmd/{diag,clean,clean_items,background,disk,schedule}.rs` | direct use of `ddc_core::sys` (iopolicy, rlimit, procs, signals, statfs, term) |
| `ddc-loadgen/src/{isolate,build,run,measure,target,verify}.rs` | direct use of `ddc_core::sys` (procs, usage, statfs, rlimit); the property tests of DDC-005's story "Coverage floor and property tests for the safety invariants" stand on it |
| `conda.rs`, `agents/{mod,claude,codex,running}.rs`, `free/probe.rs`, `disk/conda.rs`, `exec/{mod,preflight}.rs`, `report/volumes.rs`, `stats.rs` | `crate::sys` and `MetadataExt`; found by the planner and by the failed Linux and Windows `check` jobs of Gate run 41035955267 |
| `ddc-loadgen/src/build.rs`, `target.rs` and seven more | `rustix::` (`*at` tree builder), `std::os::unix`, `std::os::fd`, `cfg!(target_os…)`; every CLI test imports `ddc_loadgen::isolate`, so loadgen must compile on Windows too |
| `ddc-core/tests/common/`, `in_use.rs`, `clean_safety.rs`, `user_data.rs` and the other core test files (11 in all), and 10 CLI test files | `std::os::unix` in the test harness |
| `ddc-loadgen/tests/{guard,hostile,properties}.rs` | `std::os::unix` permission bits and symlinks; loadgen's own tests |
| `ddc-loadgen/tests/{isolation,profiles,order}.rs` | no Unix imports, but `generate().unwrap()` reaches the tree builder |
| `ddc-core/src/conda.rs`, `report/backups.rs`, `report/contents.rs` (in-source `#[cfg(test)]` modules) | `std::os::unix` and `libc::` in unit tests, which AC3 covers because they sit under `src/` |

## Decomposition

- [ ] `ddc-core/src/platform/{mod,macos,linux,windows}.rs` (or folders). `sys/` moves under `platform::macos`. Linux and Windows get stubs that compile and return `Unsupported` for each capability, with `SUPPORTED = false`.
- [ ] Every row of the table above moves behind `platform`, or behind `crate::platform` (`ddc-cli/src/platform/`) for `ddc-cli`. Shared code calls only the seam.
- [ ] The test harness: helpers that need Unix move behind `cfg(unix)` helpers in `tests/common`. Tests that are macOS-only by nature are marked `cfg(target_os = "macos")` with a one-line reason. Platform-neutral tests (rules, plan, report on fixtures, property tests) carry no `cfg`. Tests of a gated entry point (`report::scan*`, `scan::walk`, `disk::measure*`, `exec::execute`, `preflight`, `free::run`) start with `if common::unsupported("…") { return; }` (ADR "Platform seam: one platform module per OS with declared capabilities", §9). CLI integration tests that run `scan`, `clean`, `du`, `disk`, `schedule`, `notify` or `diag` skip the same way when `ddc_core::platform::SUPPORTED` is false.
- [ ] `make tools` adds the targets (`rustup target add x86_64-unknown-linux-gnu x86_64-unknown-linux-musl x86_64-pc-windows-msvc`). The gate's full tier runs `cargo clippy --workspace --all-targets --target <t> -- -D warnings` for each (`make cross-check`), or CI runs it if a warm run exceeds step 8's time budget: clippy runs `check` and also fails on the per-target unused imports a `cfg` move leaves, which would otherwise first break stage 2's native jobs.
- [ ] Before and after: `make golden-check` (DDC-005 story "Synthetic load trees: large, wide, deep and hostile"), plus the full macOS test suite.

## Implementation Plan

The how is ADR [[adr/platform-seam-one-platform-module-per-os-with-declared-capabilities|Platform seam: one platform module per OS with declared capabilities]], §1, §2, §5, §8 and §9. Each step below is its own commit. Each passes `make gate` on macOS with `golden-check` unchanged; this story never runs `golden-record`.

Decided 2026-10-30, from the planner's findings:

- **A third platform folder, `ddc-loadgen/src/platform/`.** ADR §2 already allows loadgen a `crate::platform`. Every CLI test imports `ddc_loadgen::isolate`, so loadgen must compile on every target. AC3 names the folder.
- **The forbidden list widens to `std::os::` and every `cfg` form.** `scan/reader.rs` uses `std::os::macos` and loadgen uses `std::os::fd` and `cfg!(target_os…)`, which the old tokens missed.
- **Mirrored submodules.** Shared code calls `platform::<module>::<item>` (`platform::procs::…`, `platform::signals::…`), not ADR §1's flat `platform::statfs(path)`. It is a mechanical rename from `sys::`, and the signature pins stay readable. `mod macos` stays private, so the compiler refuses `platform::macos::` outside the folder.
- **`op_stats` is not OS code.** It moves to a shared `platform/op_stats.rs`, because it needs `unsafe` (`GlobalAlloc`, `no_mangle`).
- **`file_id` fails closed.** It returns `io::Result<(u64, u64)>` and the stubs return `Err`, never `(0, 0)`: identical ids would make every guard and identity check compare equal.

Steps:

1. **Pure move.** `git mv` `sys/*` to `platform/macos/` (`platform/macos.rs` becomes `platform/macos/mod.rs`, and `sys/mod.rs`'s module declarations and doc merge into it). `platform/mod.rs` declares `pub mod op_stats;` unconditionally and `platform/op_stats.rs` carries `#![cfg(feature = "op-stats")]` as its inner attribute, the mechanism ADR §8 uses for `tests/unported.rs`, so `platform/mod.rs` names no feature but `simulate-unported` and the switches test stays as it is and `op_stats` to `platform/op_stats.rs`. Rewrite every `crate::sys::` and `ddc_core::sys::` path. Fix the two structure tests that name these paths (`structure.rs:200`, the `op_stats` path, and `:334`, `the_platform_switches_are_literals`), and change `make asan`'s `--lib sys::` filter to `platform::`. `the_platform_switches_are_literals` widens with the folder: it walks every `.rs` under `platform/` for the environment allowlist (`mod.rs` and `types.rs` read nothing; the others use the allowlist, and `HOME` is read only in `folders`), and keeps its literal-switch check on `macos/mod.rs`, `linux.rs` and `windows.rs`. Code outside the folder writes `ddc_core::platform::…` in full or aliases it; a bare `use ddc_core::platform;` is never written where a `crate::platform` exists.
2. **Types and pins.** `platform/types.rs` takes the shared types (`ProcInfo`, `VolumeStats`, `Usage`, `EntryStat`, and the listing types from `scan/reader.rs`, re-exported there so callers do not change). `read_error_kind(errno)` and the errno constants are per-OS pinned items in `platform::errno`, never in `types.rs`: `EDEADLK` is 11 on macOS and 35 on Linux. They keep the raw `i32` matches (`EDEADLK` is Dataless; never route through `io::ErrorKind`). `BulkReader` becomes `scan::reader::SystemReader`. Pin every platform item with `const _: fn(…) = imp::…`, and update the `platform/mod.rs` module doc that still shows the flat `imp::statfs` pin.
3. **Shared core code off `std::os` and `libc`.**
   - New `platform::{meta, process, trash, folders}` modules and `fsops::open_nofollow_nonblock`. `platform::meta` holds `file_id`, `link_count`, `alloc_bytes`, the mode items `add_owner_rwx` and `is_executable`, and the flags item `st_flags`.
   - Path bytes via `OsStr::as_encoded_bytes`.
   - `meta::file_id` uses std's `dev()` and `ino()`, the same widening as `bulk.rs`.
   - `SystemTrash` keeps `DeleteMethod::NsFileManager`.
   - `env.rs` gets `HOME` and the `~/Library` defaults from `folders`, in today's order.
   - The in-source test modules of `conda.rs`, `report/backups.rs` and `report/contents.rs` use `#[cfg(test)]` helpers from `platform`, or move to `tests/`.
   - Run `make coverage`, because the per-file floors of `exec/*` shift.
4. **Stubs.** One shared body in `platform/unported.rs`, used by `linux.rs` and `windows.rs`, which keep their literal switches. Stubs never call `fs::read_dir` and never read the environment:
   - `io::Result` items return `Unsupported`;
   - the two setters `init_process` calls return `Ok`, so `rules` stays quiet;
   - plain values are neutral;
   - the in-use stubs (`procs`) are intentionally empty, which fails open. They are unreachable: `require_supported` refuses in `execute`, `preflight` and `free::run`, and the dispatcher's capability check refuses before them. The Windows story inherits the signature and must fill it before it sets `SUPPORTED`.

   The `platform/mod.rs` test that names `libc::ENOSYS` uses a literal or moves behind the macOS module. `libc` and `trash` become macOS-only dependencies; confirm `cargo machete` reads target-specific dependencies without a false positive (or add them to its ignore list with a reason). `make cross-check` exists, and `-p ddc-core --lib` is green on all three targets.
5. **ddc-cli.**
   - `ddc-cli/src/platform/`: `launchd.rs` moves there (shared; only `schedule` reaches it, and `gate.rs` refuses that without the capability).
   - `Osascript` becomes `platform::SystemNotifier`, and the `Notifier` trait stays.
   - `init_process` moves into the folder with its warnings unchanged.
6. **ddc-loadgen.** `src/platform/{mod,unix,windows}.rs` takes the `rustix` `*at` builder, the byte-path conversions, `exit_signal` and the uid and mode checks. `rustix` becomes a `cfg(unix)` dependency. On Windows `build()` returns `Unsupported`. Loadgen's `guard`, `hostile` and `properties` tests are `#[cfg(unix)]` (they import Unix APIs); `isolation`, `profiles` and `order` reach the builder through `generate` and skip at run time through one helper when it returns `Unsupported`, so they run unchanged once the Windows story gives `build()` a body.
7. **Test harness (ADR §9).**
   - `tests/common` keeps one `TreeBuilder` signature on every OS. Its Unix-only bodies panic elsewhere, and only gated tests that have already skipped reach them.
   - Every test of a gated entry point starts with `if common::unsupported("…") { return; }`.
   - A new `ddc-cli/tests/common` skips the CLI tests of gated commands on `!SUPPORTED`.
   - Truly macOS-only tests get `cfg(target_os = "macos")` with a reason.
   - Ungated tests that need Unix permission bits or symlinks (`tests/agents.rs:301`, `tests/disk_free.rs:1033`, and loadgen's `guard`, `hostile` and `properties`) get `#[cfg(unix)]` with a one-line reason. This is the third category beside "skip on `!SUPPORTED`" and "macOS only".
   - 19 of the 20 CLI test files import `ddc_loadgen::isolate`; they compile on every target through loadgen's folder.
   - `interrupt_cli.rs`'s own `unsafe libc::kill` becomes `platform::signals::signal_group`.
   - One sweep with `--features ddc-cli/simulate-unported` shows every test passes or skips, as evidence for the reviewer. It is not a gate step.
8. **Enforcement.**
   - A structure test in `ddc-cli/tests/structure.rs` enforces AC3 and AC4 by reading the source.
   - `cross-check` (clippy, `-D warnings`) joins `make gate`, and `make tools` adds the targets.
   - The native `check` jobs in `gate.yml` gain `--all-targets` and also run `cargo test --workspace --locked` (nextest where installed); they stay `continue-on-error` until stage 2 of story "Build matrix: every target built, tested and packaged from one tag". Their pass and skip counts are recorded as evidence.
   - Measure the gate's added time. `cross-check` stays in the pre-push gate if a warm run adds about 90 s or less; otherwise it runs in CI only, and AC1 is met there.
   - `make require-tools` checks the three targets are installed.
   - Vault: amend the Rust core ADR's `unsafe` rule from `sys` to `platform/macos/` and `platform/op_stats.rs`, and replace DDC-005's `sys/` code_refs.
   - The Makefile and `gate.yml` edits rebase onto DDC-005 story "Gate by what changed: load at night, one gate per release, app-only pushes skip Rust" if it lands first.
9. `projectstore:reviewer` for AC5.

Risks to macOS behaviour, each with a guard:
- **Device-id widening.** A different widening of `dev` would turn every entry into another filesystem. Guard: `golden-check`.
- **Signal handling.** The errno save and restore in the signal handler, and the spawn, group, drain, clear order in `exec/system.rs`. Guard: `interrupt_cli`, `scan_cli`.
- **No new `stat`s.** None may appear in the process table, because a `stat` there could raise a TCC prompt. Guard: the reviewer reads the `procs` diff line by line.
- **Allocated-size sources.** Bulk's `ATTR_FILE_ALLOCSIZE` and std's `st_blocks * 512` stay distinct.
- **Directory-read counter.** `stats::add(DirEntriesRead)` must survive the move. Guard: the complexity guard.

## Acceptance Criteria

- [ ] `make cross-check` (`cargo clippy --workspace --all-targets --target <t> -- -D warnings`) passes for `x86_64-unknown-linux-gnu`, `x86_64-unknown-linux-musl` and `x86_64-pc-windows-msvc`, in the gate or, past the time budget in step 8, in CI.
- [ ] On macOS the full test suite passes, and the golden outputs are byte-identical: `make golden-check` passes and `make golden-record` is never run by this story.
- [ ] No file under `src/` outside `ddc-core/src/platform/`, `ddc-cli/src/platform/` and `ddc-loadgen/src/platform/` references `libc::`, `rustix::`, `windows_sys::`, `windows::` or `std::os::` (any submodule), nor a `cfg`, `cfg!` or `cfg_attr` on `target_os`, `target_family`, `unix` or `windows`; and no file in `ddc-cli` outside `ddc-cli/src/platform/`, nor in `ddc-loadgen`, references `ddc_core::sys::` or `ddc_core::platform::macos::`, enforced by a gate check (a grep, or a `ddc-shape` layer rule once DDC-006 has it).
- [ ] `unsafe_code` is denied everywhere; `#![allow(unsafe_code)]` appears only on `ddc-core/src/platform/macos/mod.rs` and `ddc-core/src/platform/op_stats.rs`, and nowhere in `tests/`.
- [ ] `projectstore:reviewer` confirmed the move preserved behaviour.

## Final Summary

## Technical Notes

- Do this before DDC-006's story "Burn down the baseline: split long files and functions". Both touch `scan/walker.rs`, `exec/*` and `disk/*`, so splitting first would mean doing the work twice.
- `sys/` becomes `platform/macos/`. DDC-006's story "Clippy at full strength: curated lints with reasons" names `sys/` (`undocumented_unsafe_blocks`, `#![allow(unsafe_code)]`); whichever lands second uses the new path.
- Prefer `rustix` (safe syscall wrappers, already in `Cargo.lock` transitively) for Linux, and `windows-sys` (also in the lock file) for Windows.

## Dependencies

- Story "Platform seam: ADR and capability model".
- DDC-005 stories "Mandatory gate before every commit" and "Coverage floor and property tests for the safety invariants" (safety net).

## Attachments

-

---

*Last updated: 2026-10-30*

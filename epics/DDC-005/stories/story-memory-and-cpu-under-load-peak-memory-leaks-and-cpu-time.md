---
type: story
id: "story-memory-and-cpu-under-load-peak-memory-leaks-and-cpu-time"
epic: "DDC-005"
title: "Memory and CPU under load: peak memory, leaks and CPU time"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [rust, testing, performance, memory, cpu]
code_refs: []
specs: []
started_at: "2026-10-28T20:40:25.833Z"
closed_at: null
plan_updated_at: "2026-10-28T20:40:25.833Z"
---

# Memory and CPU under load: peak memory, leaks and CPU time

| Field | Value |
|---|---|
| **Epic** | [DDC-005](../epic.md) |
| **Status** | in-progress |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

Nobody knows how much memory a scan of a million entries takes, whether memory is freed, how much CPU a background scan burns, or whether anything spins while it waits. The process is short-lived: the app spawns the CLI per scan and per clean. So here a memory problem means one of two things:
- peak memory grows faster than the input, or carries a large constant per entry;
- allocations are still live at exit, which shows a real leak in FFI or caching code.

A CPU problem means one of:
- user and system time out of proportion to the work;
- threads busy while waiting;
- a background scan that uses more threads than configured.

This story builds the load harness, measures each of these on the generated trees, sets limits, writes the results to dated files, and documents how to profile a regression.

**Isolation comes first.** The CLI reads and writes real user state by default: the data directory under `~/Library/Application Support/DeepDriveCleaner`, `settings.json`, the last scan, the lock, LaunchAgents and notifications. `scan` also opens `<home>/Library/Application Support/com.apple.TCC/TCC.db` to report Full Disk Access (`report::full_disk_access`); only the fake home keeps that read off the real file. A load run must touch none of it. Every invocation from the harness:
- runs with a cleared environment, `PATH` empty and `HOME` set to the profile's home from the manifest (a fake home inside the generated tree) or to the layout's empty home, which `target::prepare` creates before the first call because `Env::from_process` canonicalizes it. It is never the real home, compared with `getpwuid(getuid())` by path and by device and inode, not with the parent's `$HOME`;
- passes `--data-dir` inside the generated layout, created empty by the harness (so no stale lock can exist) with a `settings.json` it wrote;
- passes `--no-notify` whenever it passes `--background`;
- targets only a tree the generator created: its disk image, or its marked temporary folder for S trees.

The harness refuses to start if `HOME` resolves to the real home, if the data directory is missing from the command line or lies outside the layout, if `--background` comes without `--no-notify`, if the target fails the generator's identity check, or if a `clean --items` that trashes anything is aimed at a target other than an attached image. An S tree lives in `$TMPDIR` on the home's volume, so a trash there would land in the owner's `~/.Trash`; on the image it lands in the image's own `.Trashes` and leaves with the detach.

No tool runs from the harness. Three profiles (`users`, `mixed`, `system-root`) hold findings whose removal is `native` or `native-only` (Docker, podman, `go clean` and the like). The harness leaves them out of the `--items` list, asserts the list it passes holds only `trash` and `delete` actions, and fails if a clean report names a native item or a tool. `PATH` empty does not stop tool discovery alone (`Env::tool_dirs` adds Homebrew and Cargo folders, and `SystemCommands::new` always searches `/usr/bin`). Two barriers keep tools out:
- the items list keeps every item's own action to `trash` or `delete`, and before the real clean the harness runs `clean --items --dry-run --json` on the same list and refuses the clean if any prediction would run a tool (preflight finds tools but never runs them);
- `exec::execute` counts local Time Machine snapshots with `/usr/bin/tmutil listlocalsnapshots /` when its caller passes no count, and `clean --items` passes none today. A hidden `--local-snapshots <N>` option on `clean --items` (one line in `cmd/clean_items.rs`, passed into `ExecOptions::local_snapshots`) lets the harness pass `0`, so no tool runs and `clean --items`'s CPU time is the CLI's own (`wait4` would fold a reaped `tmutil`'s usage into the child's).

`clean --dry-run` with the default plan runs no tool (it returns before `SystemCommands` is built), and `disk --system-root` resolves its tools under the tree.

## Decomposition

- [x] Measurement primitives in `ddc-core/src/sys` (the workspace keeps `unsafe` there; `rustix` 1.1 has no `wait4` and no `getrusage` on macOS):
  - `sys::usage`: `wait4` per child with its `rusage` (user, system, peak RSS normalised to bytes: bytes on macOS, KiB on Linux), `RUSAGE_SELF`, and the calling thread's CPU clock (`CLOCK_THREAD_CPUTIME_ID`);
  - `sys::procs::thread_count(pid)` (`proc_pidinfo` `PROC_PIDTASKINFO`, `pti_threadnum`);
  - for the TSV header: the load average (`getloadavg`) and the machine model (`sysctl hw.model`).
- [x] The harness: a `ddc-loadgen run <profile> --entries N --seed S --into <dir> --device D --cli <path>` subcommand, with the isolation rules above as a pure decision (`isolate.rs`, like the guard's `target.rs`) and a test that it refuses each unsafe setup. `scripts/load-image.sh` calls it instead of `build --verify --brief` when `LOAD_CLI` is set, so the image's lifecycle stays in one place. `run` builds the tree and verifies it against the manifest before it measures.
- [ ] `make load` (`LOAD_PROFILE`, default `all`; `LOAD_SIZE`, default M): build the release CLI and generator, then for each profile run the release build of each command on its tree, in this order (every command that reads the tree comes before the one that changes it):
  - `scan --json`;
  - `clean --dry-run`;
  - `scan --background --no-notify` (no `--progress`: the two conflict);
  - `disk --system-root <tree>`, on the `system-root` profile only;
  - `clean --items <list> --local-snapshots 0`, once per tree, on the image only, after a `clean --items --dry-run --json` of the same list whose predictions run no tool; the harness refuses the clean otherwise. The clean passes when every listed item ends trashed or deleted (a refused or failed outcome fails the profile and is recorded with its reason; with `--json` the CLI exits 0 either way), the entries in `<mount>/.Trashes/<uid>` equal the trashed items, and no outcome is native. Delete only inside the generated tree.

  The first four run `LOAD_REPEAT` times (default 3); every run is a row, and `compare_load.py` takes the median. Record, per command and run:
  - wall time;
  - user and system time and their sum, from `wait4` on that child (not `RUSAGE_CHILDREN`, whose peak RSS is the maximum over every child so far);
  - peak RSS (`ru_maxrss`);
  - the sampled peak thread count (`pti_threadnum`, polled every 10 ms; rusage has none);
  - entries and the exit code.

  The Gate workflow's `load` job becomes `make load LOAD_SIZE=M LOAD_LIMITS=record LOAD_REPEAT=1`: one image, the manifest check plus the measurements. `run` builds into `--into`: on an image the detach removes the tree; a marked temporary folder stays until `make loadgen-clean`, and the tests wrap theirs in `target::Scratch`, removed on drop. The data folder is shared by the repeats, so run 1 of `clean --dry-run` finds no `last-scan.json` and later runs do; with `LOAD_REPEAT=1` CI measures the first case. The clean's list is every plan item whose action is `trash` or `delete`, selected or not, without incomplete findings and without rules that have an owning app (a run of the owner's browser skips those).
- [ ] Memory limit, per `(command, profile)`: peak RSS ≤ base + k bytes per entry. Base and k are measured on the reference machine from S and M runs (base from S, k from the slope between S and M), then set with 25% headroom. `LOAD_LIMITS=enforce` (the default of `make load`) fails over a limit; `record` (CI) writes the numbers and asserts nothing. The harness records the 1-minute load average at the start and end in the TSV header, and falls back to `record` for the CPU limit when the start value is above a quarter of the CPU count, so a busy machine records CPU and does not judge it. The memory limit is enforced regardless of load: peak RSS does not depend on it. The memory scaling slope (≤ 1.2) belongs to story "Complexity guard: operation counts and scaling exponents", which reuses this harness and its columns.
- [ ] Leak check: a `ddc-loadgen leaks --cli <path>` subcommand runs, for every profile at S in a marked temporary tree, `leaks -quiet -atExit -- target/release/deepdrivecleaner scan <tree> --json --data-dir <data>` (the cargo release build, not the notarized app copy, which hardened runtime keeps `leaks` out of), with the harness's cleared environment. A profile passes on exit 0 and a parsed `0 leaks for 0 total leaked bytes` line. `-atExit` sets `MallocStackLogging=lite`, so a leak report already carries its stacks; an exit above 1 or a missing line (a child killed by a signal) is a failure. `make leaks` runs in `make gate`, after `release-check`, reusing its release binaries; its cost (11 S builds and 11 `leaks` runs) is recorded against the epic's pre-push budget of 5 minutes. The first `make leaks` on the reference machine is the owner's run; it enters `make gate` after that run. Symbol exclusions are allowed only with a written reason; the list starts empty.
- [ ] AddressSanitizer: an `asan` job in the nightly workflow, on a dated nightly toolchain, with `RUSTFLAGS=-Zsanitizer=address`, `ASAN_OPTIONS=detect_leaks=0` (LeakSanitizer does not run on macOS) and `--target aarch64-apple-darwin` (so proc-macros are not instrumented). It runs the `sys::` unit tests of `ddc-core` and the generator's profile and hostile tests, which scan an S tree of every profile in process.
- [ ] Heap profile: a `dhat-heap` cargo feature of `ddc-core` (`dhat` crate) with a global allocator, started first in the CLI's `main` through `ddc_core::heap::start()`, which is a no-op without the feature (`ddc-cli` cannot test a dependency's feature). It cannot be combined with `op-stats` (`compile_error!`). `make heap-profile` builds into `target/dhat` with a `profiling` cargo profile (release plus line tables), so it never replaces `target/release/deepdrivecleaner`, and writes the allocation sites of one `scan` at M (L when asked). The top 10 program points by bytes live at the global peak, and by total bytes, each divided by entries, are written into this story.
- [ ] CPU budget, per `(command, profile)`: (user + system) / entries ≤ a measured limit with headroom, the median of the repeated runs, enforced on the reference machine under the same `enforce`/`record` rule as memory. `wide-root`'s CPU limit is re-baselined when story "Complexity guard: operation counts and scaling exponents" removes its `dir_has_any` re-read (an 18 s scan at M).
- [x] Threads: the background scan's walk options come from one named function in `cmd/background.rs`, and a `ddc-cli` test checks it gives `threads: Some(3)`. The function runs no program, so `structure.rs::the_background_scan_runs_no_program_itself` still holds. A walker test runs `scan::walk_with_reader` with `threads: Some(3)` through a wrapping `DirReader` and counts distinct `ddc-walk-*` threads: at most 3. Low I/O priority is the LaunchAgent's job: a test checks the CLI-generated plist (`launchd.rs`) still carries `LowPriorityIO` and `Nice 10`. The app's scheduler plist is DDC-002's.
- [x] No busy waiting at the real idle points:
  - the CLI's progress ticker between ticks (`ddc-cli/src/progress.rs`, `park_timeout`, `TICK` 50 ms) around a 2 s blocking step;
  - `--progress json` between events, with progress that does not change for 2 s;
  - the wait for a native command in `exec/system.rs` (`try_wait` + `park_timeout(50 ms)`, up to `NATIVE_TIMEOUT`), with `/bin/sleep 2` as the command.

  Each is measured on the waiting thread's own CPU clock (`CLOCK_THREAD_CPUTIME_ID`), so other tests in the same process do not pollute it. For each, CPU time over the idle stretch is < 1% of wall time. (`OPEN_FILE_CONFIRM` in `exec/guard.rs` is one plain `sleep`, with nothing to measure.)
- [ ] `bench/load-<date>-<size>.tsv` in a fixed column order: a `#` line with the date, commit, machine model, CPU count, load average at start and end, limits mode and seed, then a header row: `profile size seed entries command run wall_ms user_ms sys_ms cpu_ms max_rss_bytes threads_max exit`. `run` writes its rows to `target/load/<size>-<profile>.tsv`; `make load` concatenates them under one `#` line into `bench/load-<date>-<size>.tsv` only in `enforce` mode, and refuses to overwrite an existing file of that name. A `record` run (CI, or a busy reference machine) leaves its file in `target/load`; CI uploads it as a workflow artifact. `bench/compare_load.py` compares CPU time (not wall time) and peak RSS, per `(profile, size, command)` median, against the newest earlier file of the same size, and flags any metric more than 15% worse. Its tests run in `make gate`.
- [ ] After a run, `make load` checks the owner's state by modification time only, never by listing: the data directory `~/Library/Application Support/DeepDriveCleaner` and the files in it, `~/Library/LaunchAgents` and `~/Library/Logs/DeepDriveCleaner`. A change names the path and the time, since the owner's app may scan on its own schedule meanwhile. After `clean --items`, it counts the entries in `<mount>/.Trashes/<uid>` against the clean report's trashed items. The real Trash is never read.
- [ ] Runbook "Profiling the core" in the vault's `ops/` (`type: runbook`):
  - flamegraph with `samply` or Instruments, on the `profiling` build;
  - heap profile with `dhat`;
  - `leaks`;
  - what to do when `compare_load.py` flags a metric, telling a regression from a stale baseline (a changed profile, catalog or reference machine), and how to re-baseline;
  - `leaks` exclusions: when an OS update surfaces a system-library allocation, how to add one with its written reason.

  Story "Complexity guard: operation counts and scaling exponents" then adds the section on counters.

## Implementation Plan

Planned with `projectstore:planner` on 2026-10-28. Five commits, each passing the gate on its own, then the runbook.

1. **Measurement primitives and the in-process checks** (all in `gate-fast`).
   - `ddc-core/src/sys/usage.rs` (`Usage`, `wait4`, `self_usage`, `thread_cpu_time`) and `sys::procs::thread_count`, with their own tests: `wait4` on `/bin/sleep 0` returns success and a non-zero peak RSS; the thread count rises with spawned threads.
   - The idle tests: the ticker and JSON reporter in `ddc-cli/src/progress.rs`'s tests, the native wait in `ddc-core` (`exec/system.rs` tests or `tests/idle.rs`).
   - The background walk options extracted and tested; the walker thread-count test; the plist's `LowPriorityIO` and `Nice`.
2. **The harness, `make load` and the TSV.**
   - `ddc-loadgen`: `isolate.rs`, `measure.rs` (spawn with stdout and stderr to files, poll `thread_count` and `wait4(WNOHANG)`, never `Child::wait`, which would reap the child first), `items.rs` (selected `trash`/`delete` plan items with `dev`, `ino` from `symlink_metadata` and the class from the manifest's findings, validated `ListedItems`), `table.rs` (the columns, shared later with `make scaling`), `run.rs`; `Cmd::Run` in `main.rs`.
   - The hidden `--local-snapshots` option of `clean --items`.
   - `scripts/load-image.sh` (`LOAD_CLI` branch), Makefile `load`, the CI `load` job switched to it. In this commit `make load` only records: there are no limits yet, and `LOAD_LIMITS` is accepted and ignored with a note on stderr. `enforce` becomes the default in commit 4, with the baseline.
   - `.config/nextest.toml`: `binary(load_cli)` joins the `load` group's filter.
   - Tests: `ddc-loadgen/tests/isolation.rs` (decision table), `ddc-cli/tests/load_cli.rs` (an S `projects` tree through the harness: non-zero user time and RSS, exit 0, a full row; no native item in the `users` list), in the nextest `load` group.
3. **Leaks in the full gate, AddressSanitizer nightly.** `ddc-loadgen leaks` with a tested parser of `leaks` output (its trees are `target::Scratch`), `make leaks`, the nightly `asan` job. The owner runs the first `make leaks` on the reference machine; `make leaks` joins `make gate` after it.
4. **Limits and the comparison.** `ddc-loadgen/src/limits.rs` (per `(command, profile)` base, k and CPU per entry; `enforce`/`record` and the load-average fallback; tests at and over the limit), `bench/compare_load.py` and its tests. Then the reference-machine baseline on a quiet machine (the owner's), and the throwaway ballast branch for AC 4, never merged.
5. **Heap profile.** The `dhat-heap` feature, `ddc-core/src/heap.rs`, the `profiling` cargo profile, `make heap-profile`, `bench/dhat_top.py`; the top 10 into this story. `make deny` early, for the new crates in `Cargo.lock`.

Then the runbook, reviewed by the critic.

## Acceptance Criteria

- [ ] The harness refuses a run with the real `HOME`, a missing or outside `--data-dir`, `--background` without `--no-notify`, a target that fails the identity check, a trash clean on a temporary target, and a clean whose dry run predicts a tool run (tests). After a full `make load`, the owner-state check reports, by modification time only, every change during the run to the data directory, its files, `~/Library/LaunchAgents` and `~/Library/Logs/DeepDriveCleaner`, with path and time. Met by one run with the menu bar app quit that reports no change (owner: app quit); a run with the app open may report `last-scan.json` and is advisory. Every listed item ends trashed or deleted, and the entries in `<mount>/.Trashes/<uid>` match the trashed items; the real Trash is never read.
- [ ] `make load` runs on the reference machine and writes `bench/load-<date>-M.tsv` with all listed columns for every profile and command at M.
- [ ] `leaks -atExit` reports 0 leaks for every profile at S, in `make gate` (`make leaks`), after the owner's first run on the reference machine. The nightly AddressSanitizer job is green.
- [ ] Per-`(command, profile)` memory and CPU limits exist in the harness, and an M run on a quiet reference machine passes them in `enforce` mode (owner: quiet machine, for the CPU baseline). A deliberately added allocation retained per entry, at least twice the headroom (a throwaway branch that keeps a 1 KiB buffer per entry in a long-lived `Vec`), fails the memory limit.
- [ ] The idle-point tests show < 1% CPU on the waiting thread.
- [ ] `compare_load.py` flags a seeded 20% regression, passes an unchanged rerun, and ignores a wall-time-only change (tests).
- [ ] The runbook "Profiling the core" exists and was reviewed by the critic.

## Final Summary

## Technical Notes

- New `sys` wrappers: `wait4` and `getrusage` (`libc`), `proc_pidinfo` `PROC_PIDTASKINFO`, and the thread CPU clock. `rustix` 1.1 has no `wait4` and no `getrusage`, and `unsafe` stays confined to `ddc-core/src/sys`, as with `sys::procs::real_home()` for the generator.
- `ru_maxrss` is in bytes on macOS and in KiB on Linux. `sys::usage` normalises it; DDC-007 reuses this harness.
- The harness writes the child's stdout and stderr to files, not pipes: `scan --json` at M is megabytes and a full pipe would block the child.
- `leaks` needs the process to exit normally. A run killed by a signal is a test failure, not a leak report. Xcode's `leaks` (behind the `/usr/bin/leaks` shim) holds `com.apple.system-task-ports.read`, so on the ad-hoc-signed cargo build it raises no Developer Tools Access dialog and needs no `sudo` or TCC grant. Its report shares stdout with `--json`, so the harness judges by exit code and the parsed summary line.
- `clean` reaches Objective-C through the `trash` crate (`NSFileManager`); autorelease-pool objects may show up in `leaks`. `scan` never calls it, so the gate's check stays strict for `scan`. `clean --items` runs under `leaks -autoreleasePools` separately, informational, on an image only, never on an S temporary tree.
- `dhat` records a backtrace per allocation: an instrumented scan is many times slower than the 51.6 s plain L `mixed` scan, hence M by default.
- A clean on a generated tree goes through the real executor. Anything trashed goes to the image's `.Trashes` and leaves with the detach. Without `--local-snapshots`, the executor would run `/usr/bin/tmutil listlocalsnapshots /` (read-only, 30 s timeout) to count snapshots; the harness passes `0`.
- If `NSFileManager` refuses to trash on the `-nobrowse` image, every trash item ends failed and the clean's pass criterion catches it.

## Dependencies

- Story "Synthetic load trees: large, wide, deep and hostile".
- Story "Mandatory gate before every commit".

## Attachments

-

---

*Last updated: 2026-10-28*

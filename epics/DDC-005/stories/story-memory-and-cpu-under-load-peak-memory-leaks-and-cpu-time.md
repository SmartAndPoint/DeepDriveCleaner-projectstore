---
type: story
id: "story-memory-and-cpu-under-load-peak-memory-leaks-and-cpu-time"
epic: "DDC-005"
title: "Memory and CPU under load: peak memory, leaks and CPU time"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [rust, testing, performance, memory, cpu]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Memory and CPU under load: peak memory, leaks and CPU time

| Field | Value |
|---|---|
| **Epic** | [DDC-005](../epic.md) |
| **Status** | planned |
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

**Isolation comes first.** The CLI reads and writes real user state by default: the data directory under `~/Library/Application Support/DeepDriveCleaner`, `settings.json`, the last scan, the lock, LaunchAgents and notifications. A load run must touch none of it. Every invocation from the harness:
- runs with `HOME` set to the profile's home from the manifest (a fake home inside the generated tree) or to an empty `<scratch>/home`, created before the first call because `Env::from_process` canonicalizes it. It is never the real home, compared with `getpwuid(getuid())`, not with the parent's `$HOME`;
- passes `--data-dir <scratch>/data`, with a `settings.json` the harness wrote;
- passes `--no-notify` whenever it passes `--background`;
- targets only a tree the generator created: its disk image, or its marked temporary folder for S trees.

The harness refuses to start if `HOME` resolves to the real home, if the data directory is missing from the command line, or if the target fails the generator's identity check.

No tool runs from the harness. The `--items` list is built from the manifest and holds only `trash` and `delete` actions. The harness fails if a plan on a load tree contains a `native` or `native-only` action (Docker, podman, `go clean` and the like).

## Decomposition

- [ ] The harness (`ddc-loadgen run`, or a script next to it) with the isolation rules above, and a test that it refuses each unsafe setup.
- [ ] `make load`: for each profile at M (and L when asked), run the release build of each command on a generated tree:
  - `scan --json`;
  - `clean --dry-run`;
  - `clean --items <list>` on the image;
  - `disk --system-root <tree>` on the `system-root` profile.
  
  Delete only inside the generated tree. Record, per command:
  - wall time;
  - user and system time;
  - peak RSS (`getrusage` `ru_maxrss` of the child);
  - thread count;
  - entries.
- [ ] Memory limit: peak RSS ≤ base + k bytes per entry. Measure k on the reference machine first, then set the limit with 25% headroom. It is checked on the reference machine only. CI records the number and asserts nothing absolute. The memory slope from story "Complexity guard: operation counts and scaling exponents" stays ≤ 1.2.
- [ ] Leak check: `leaks --atExit -- target/release/deepdrivecleaner scan <tree> --json --data-dir <scratch>/data` (the cargo release build, not the notarized app copy, which hardened runtime keeps `leaks` out of). It reports 0 leaks for every profile at S, in the full gate. `MallocStackLogging=1` gives stack traces when it fails.
- [ ] AddressSanitizer: a nightly-toolchain job (`-Zsanitizer=address`) runs the `sys/` tests and an S scan of every profile, in the nightly workflow.
- [ ] Heap profile: a `dhat-heap` cargo feature (`dhat` crate) writes the allocation sites of one L scan. `make heap-profile` produces it. The top 10 sites and their bytes per entry are written into this story.
- [ ] CPU budget: (user + system) / entries ≤ a measured limit with headroom, the median of 3 runs, on the reference machine only.
- [ ] Threads: a library test runs a scan with the background options and checks it uses at most 3 worker threads. Low I/O priority is the LaunchAgent's job: a test checks the CLI-generated plist (`launchd.rs`) still carries `LowPriorityIO` and `Nice 10`. The app's scheduler plist is DDC-002's.
- [ ] No busy waiting at the real idle points:
  - the CLI's progress ticker between ticks (`ddc-cli/src/progress.rs`, `park_timeout`, `TICK` 50 ms), measured in process with `getrusage(RUSAGE_SELF)` around a 2 s blocking step;
  - `--progress json` between events;
  - the wait for a native command in `exec/system.rs` (`try_wait` + `park_timeout(50 ms)`, up to `NATIVE_TIMEOUT`), with a fake command that sleeps.
  
  For each, CPU time over the idle stretch is < 1% of wall time. (`OPEN_FILE_CONFIRM` in `exec/guard.rs` is one plain `sleep`, with nothing to measure.)
- [ ] `bench/load-<date>.tsv` in a fixed column order. `bench/compare_load.py` compares CPU time (not wall time) and peak RSS, and flags any metric more than 15% worse than the previous file.
- [ ] Runbook "Profiling the core" (`ops/`):
  - flamegraph with `samply` or Instruments;
  - heap profile with `dhat`;
  - `leaks`;
  - what to do when `compare_load.py` flags a metric.
  
  Story "Complexity guard: operation counts and scaling exponents" then adds the section on counters.

## Implementation Plan

## Acceptance Criteria

- [ ] The harness refuses a run with the real `HOME`, a missing `--data-dir`, or a target that fails the identity check (tests). After a full `make load`, the owner's data directory and `~/Library/LaunchAgents` have unchanged modification times (both readable without a TCC grant). Trashed items are shown, by construction, to be under the image's `.Trashes`. The real Trash is never read.
- [ ] `make load` runs on the reference machine and writes `bench/load-<date>.tsv` with all listed columns for every profile and command at M.
- [ ] `leaks --atExit` reports 0 leaks for every profile at S, in the full gate. The nightly AddressSanitizer job is green.
- [ ] Per-entry memory and CPU limits exist in the harness, and an M run on the reference machine passes them. A deliberately added allocation retained per entry, at least twice the headroom (a throwaway branch that keeps a 1 KiB buffer per entry in a long-lived `Vec`), fails the memory limit.
- [ ] The idle-point tests show < 1% CPU.
- [ ] `compare_load.py` flags a seeded 20% regression and passes an unchanged rerun.
- [ ] The runbook "Profiling the core" exists and was reviewed by the critic.

## Final Summary

## Technical Notes

- `ru_maxrss` is in bytes on macOS and in KiB on Linux. The harness normalises it; DDC-007 reuses this harness.
- `leaks` needs the process to exit normally. A run killed by a signal is a test failure, not a leak report.
- `clean` reaches Objective-C through the `trash` crate (`NSFileManager`); autorelease-pool objects may show up in `leaks`. If the check proves noisy, keep it strict for `scan` (no Objective-C path) and run `clean` under it separately with `MallocStackLogging`.
- A clean on a generated tree goes through the real executor. Anything trashed goes to the image's `.Trashes` and leaves with the detach.

## Dependencies

- Story "Synthetic load trees: large, wide, deep and hostile".
- Story "Mandatory gate before every commit".

## Attachments

-

---

*Last updated: 2026-10-28*

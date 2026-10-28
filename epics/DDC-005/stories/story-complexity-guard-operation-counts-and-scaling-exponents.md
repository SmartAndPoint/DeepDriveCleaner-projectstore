---
type: story
id: "story-complexity-guard-operation-counts-and-scaling-exponents"
epic: "DDC-005"
title: "Complexity guard: operation counts and scaling exponents"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [rust, testing, performance, complexity]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Complexity guard: operation counts and scaling exponents

| Field | Value |
|---|---|
| **Epic** | [DDC-005](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

A wall-clock limit ("scan in ≤ 60 s") catches a slow machine as readily as a slow algorithm, and it misses a quadratic step until the input is big enough. This story catches a quadratic step by its shape, in two complementary ways:

1. **Cost counters (deterministic, in the fast gate).** The core counts the work its hot paths do. The counters measure cost, not call sites. A loop that calls a cheap function n times and a function that reads a directory of n entries both show up.
   - `dir_entries_read`: every entry returned by a directory read, wherever it happens. Counting inside the two `DirReader` implementations (`scan/reader.rs`, `sys/bulk.rs`) covers the walker and `exec/preflight.rs`. One `counted_read_dir` helper covers the direct `fs::read_dir` sites: `report/project.rs` (`last_activity`), `rules/matcher.rs`, `rules/classify.rs`, `rules/toolchain.rs`, `exec/delete.rs` and `disk/`. This is the counter that catches `aa64c83` step 2.
   - `claim_lookups`: comparisons made while matching claims to findings. This catches step 1.
   - `rule_match_attempts`.
   - `ancestor_checks`.
   - `allocations` and `bytes_allocated`: a counting `#[global_allocator]` wrapper, active only under the feature.

   A test runs the same profile at n and 4n entries and checks each counter's ratio. It fails if a counter grows faster than its declared bound: linear for most, n·log n where it is justified and written next to the counter. A quadratic step multiplies by 16 instead of 4. That is visible on S-size trees, independent of the machine, and never flaky.
2. **Scaling slopes (wall clock and memory, measured on the reference machine).** The test times scan, report, plan and the after-clean recheck at n, 2n, 4n and 8n on M and L trees. It fits the log-log slope of CPU time and of peak memory, and reports above 1.2. This catches costs that no counter sees: syscalls, locks, cache misses.

## Decomposition

- [ ] A `stats` module in `ddc-core`. Counters are relaxed atomics, or per-thread counters summed at the end, behind a cargo feature `op-stats`, which is not a default feature. Each counter is declared with its bound (`Linear`, `NLogN`) and a one-line reason. Tests read counters as deltas around the measured scan. The ratio tests run serially (a nextest `test-group` with `max-threads = 1`) and only through nextest, which runs each test in its own process; `make check` is an alias of `gate-fast`, so no gate path runs them under plain `cargo test`.
- [ ] `op-stats` and the memory story's `dhat-heap` both install a `#[global_allocator]`: a `compile_error!` refuses both at once.
- [ ] The counters above at their sites. A profile of an L `mixed` scan confirms no hot path is missed.
- [ ] Fast-gate test, in process (library calls, no harness needed): `wide-root`, `users` and `projects` at S and 4·S; the other profiles at 10³ and 4·10³ (the ratio depends on shape, not size). Assert `count(4n) / count(n) ≤ 4 · (1 + ε)` for `Linear` and the matching bound for `NLogN`.
- [ ] A third quadratic step, found while planning the load trees and not fixed by `aa64c83`: `rules/classify.rs` `dir_has_any` reads the project root for the glob `requirements*.txt` once per venv with no lock file beside it, uncached. The `wide-root` profile (10⁴ siblings, N venvs) must show it in `dir_entries_read`. The step fires when a venv sits directly under that root (parent = project root, so `[parent, project_root]` reads it twice) and no lock file is beside it. Fix it, for example with a `dir → has_lock` memo inside one `classify` call, or by recording lock-file presence in the visitor's `on_file`; both cost O(distinct directories), unlike caching whole listings. Keep the guard as its regression test.
- [ ] Regression tests for `aa64c83`. The counters are written on the current code first. Then, on a throwaway branch, each step is restored on its own: the per-claim scan over all finding paths, then the per-finding `last_activity` read. The guard must fail each time, naming the counter. Record both failing outputs in this story.
- [ ] `make scaling`: reuses the memory story's harness and column order. CPU time (user + system, median of 3 runs) and peak memory at four sizes, slope fit, and a table written to `bench/scaling-<date>.tsv`. It runs on the reference machine. CI runs it at M for information only.
- [ ] A section "Adding a counter" in the runbook "Profiling the core" from story "Memory and CPU under load: peak memory, leaks and CPU time". That story writes the runbook first.

## Implementation Plan

## Acceptance Criteria

- [ ] With `op-stats` on, every profile passes the counter-ratio test at the sizes above in the fast gate, in ≤ 15 s total, or the measured time is recorded and the sizes are lowered.
- [ ] Restoring either quadratic step of `aa64c83` alone makes the counter test fail with a message that names the counter and the measured ratio (evidence: two throwaway-branch outputs).
- [ ] Before the `dir_has_any` fix, `wide-root` at S and 4·S fails the `dir_entries_read` ratio (the output is recorded in this story); after the fix it passes.
- [ ] `make scaling` on M and L trees on the reference machine produces slopes ≤ 1.2 for CPU time and peak memory of scan and report, recorded in `bench/`.
- [ ] The release binary carries no counters: `scripts/build-dist.sh` passes no `--features`, `op-stats` is not a default feature (both asserted in `ddc-cli/tests/structure.rs`), and `nm` on the dist binary finds no `op_stats` symbol.

## Final Summary

## Technical Notes

- Pick ε so that the n·log n terms of sorting fit the bound: about 0.25 at these sizes.
- The counting allocator replaces the global allocator only under `op-stats`, so the counted numbers come from a slightly different binary than the shipped one. The peak-RSS numbers in the memory story come from the real release build.

## Dependencies

- Story "Synthetic load trees: large, wide, deep and hostile" (the trees and their manifests).
- Story "Mandatory gate before every commit".
- Story "Memory and CPU under load: peak memory, leaks and CPU time" (the isolated harness for `make scaling`, and the runbook). The in-process counter tests do not wait for it.

## Attachments

-

---

*Last updated: 2026-10-28*

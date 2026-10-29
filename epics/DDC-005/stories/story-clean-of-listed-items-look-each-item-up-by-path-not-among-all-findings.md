---
type: story
id: "story-clean-of-listed-items-look-each-item-up-by-path-not-among-all-findings"
epic: "DDC-005"
title: "Clean of listed items: look each item up by path, not among all findings"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-29
updated: 2026-10-29
external_refs: {}
tags: [rust, performance, complexity]
code_refs: ["crates/ddc-core/src/plan.rs", "crates/ddc-core/src/report/after_clean.rs", "crates/ddc-core/src/exec/preflight.rs", "crates/ddc-core/src/stats.rs", "crates/ddc-core/src/exec/report.rs", "crates/ddc-core/src/exec/mod.rs", "crates/ddc-loadgen/src/scaling.rs", "crates/ddc-loadgen/src/verify.rs", "crates/ddc-loadgen/tests/profiles.rs"]
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Clean of listed items: look each item up by path, not among all findings

| Field | Value |
|---|---|
| **Epic** | [DDC-005](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

`clean --items` (the app's Clean, and the load harness's `clean-items`) grows quadratically with the items it is given times the findings. `make scaling` at L on `mixed`, on `2af7940`, 2026-10-29 (a busy machine, 1-minute load 33 at the start): the median user CPU time of `clean-items` was 9.2, 31.7, 105.9 and 409.2 s at 125 000, 250 000, 500 000 and 10⁶ entries, a local slope of 1.95. `scan` on the same trees: 0.27 to 2.1 s, a user-time fit of 0.98. One clean of the planted items of a 10⁶-entry tree spends almost 7 minutes of CPU in the core's own loops, not in the file system.

The harness lists every planted item, about 3% of the entries (29 828 of 10⁶ in `mixed`). A clean from the app sends tens to a few hundred items, where O(items × findings) is small; the exposure is a large selection, such as the whole-disk sheet (story "Whole disk: free the checked areas in one sheet", epic DDC-002) or a user selecting a whole group. Hence p2.

The suspects are the lookups the story "Complexity guard: operation counts and scaling exponents" listed as out of its scope (runbook "Profiling the core", step 8). Each looks one listed item up among all findings, entries or lines, so a clean is O(items × findings):
- `plan.rs` `plan_listed`: `report.findings.iter().find` per item, and `report.findings.iter().any` for a `KEPT` ancestor per item.
- `exec/preflight.rs`: `trashed.iter().any(|t| item.path != *t && item.path.starts_with(t))` per item, over every would-trash item: about 30 000 × 2 500 `starts_with` at 10⁶ entries of `mixed`.
- `report/after_clean.rs`: `finding`, `at_or_under`, `trash_on`, `leave_entry`, `refresh`'s `fresh.findings.iter().find` and `finish`'s `lines.iter().any`, each called per item. (`remove_entry`'s `position` runs over at most 100 entries, `report/contents.rs`: bounded.)
- `exec/report.rs` `CleanReport::new`: `refused.iter().find` and `run.items.iter().find` per listed item, items × items.

Which of them carry the 409 s is measured first, not assumed.

## Decomposition

- [x] Profile `clean --items` and record the top frames of user time here. Done at M, not L (Technical Notes, "Profile at M").
- [ ] Index each lookup with the equality it uses today (Implementation Plan), `leave_entry` and `at_or_under` first (the profile). `plan_listed` and `entry()` compare paths byte for byte: a `HashMap<&OsStr, Vec<usize>>` keyed by `as_os_str()`, in finding order. `Edit` and `CleanReport::new` compare by component (`/a/b/` equal to `/a/b`): keyed by `Path`. The `KEPT` ancestor check is a set of the KEPT paths probed with `item.path.ancestors()`: not `AtOrAbove`, which answers the opposite question (a set path at or below a folder). `at_or_under` becomes a range of the findings sorted by `Path`'s own order, which compares by component (`/a/b/x` before `/a/b-old`); a byte-wise sort would break the range. `trash_on` stays a scan of a short sorted set of candidate indices, each re-checked against the live finding's rule and `dev` when probed, since `refresh` can change a finding's rule either way.
- [ ] Keep the cleanup safety model as it is (ADR "Cleanup safety model: risk classes, plan and confirmation"): a listed item still matches on path, `(dev, ino)`, rule, class and action; the refusal reasons (`kept`, `changed-since-scan`, the others) and the order `entry()` checks them in are unchanged. Only how a finding is found changes.
- [ ] Count every per-item lookup in a new counter, `item_lookups` (one per index probe or per comparison). It lumps hash probes, ancestor probes and range elements, unlike the one-cost-per-counter rule of `stats.rs`; every site it counts has the same scale (items × depth), so a restored items × findings scan dominates it at any n, and extend the in-process ratio test in `crates/ddc-loadgen/tests/profiles.rs` to measure `plan_listed` and the after-clean bookkeeping over the item list it already runs `preflight` on.
- [ ] Prove the test can fail: restore one per-item scan on a throwaway worktree, never committed, with its comparison counted as step 4 of "Adding a counter" in the runbook requires, and record the failing line. A scan restored verbatim, uncounted, passes the counters (the parent story measured this on `d9ad787`); the user-time fit of `make scaling` is the guard for that case.
- [ ] Judge `clean-items` in `ddc-loadgen slopes`: the run count becomes per command (`clean-items` runs once per size, since it trashes its items), and its user-time fit is judged like `scan`'s once it is linear. Its user time is seconds at n, far above the floor, and a single-run spread of ε at one end of the four sizes moves the least-squares fit by 1.5·ln2·ε / (5·(ln2)²) ≈ 0.43ε, and at both ends in opposite directions by about 0.87ε, so 10% costs 0.04 to 0.09. Without this nothing automated guards it after the fix: the counters miss an uncounted loop.
- [ ] Replace the out-of-scope paragraph of step 8 in the runbook with what was fixed.

## Implementation Plan

From `projectstore:planner` (opus), 2026-10-29, on `e933b7b`, ranked by the profile at M (Technical Notes). From M to 10⁶ items and findings grow about 10×, so an items × findings site grows about 100× and a linear one 10×: the 32.5% of `Edit` at M becomes about 83% at L. That alone does not give the measured slope: a 32.5% quadratic share at M predicts a local slope of 1.77 and a four-size fit of about 1.62, against 1.95 and 1.82 measured. The measured numbers need a quadratic share of about 73% at M, so about 40 points are not attributed by the profile: `plan_listed` (items × findings) and `CleanReport::new` (items × items) are not among the top frames at M but grow 100×, about 9% of `exec::execute` is outside delete, journal and guard, and the profile ran a profiling build on the main thread against a release build measured on all threads. Step 8 covers the case where a site is still left.

1. Tests for the two places an index could change the answer, on the current code, passing: an unknown rule under a kept ancestor gets `rule-mismatch`, not `kept`; and with several findings at one path, `plan_listed` takes the first that matches and `Edit::finding` the first not removed.
2. `item_lookups` in `stats.rs` (`Counter`, `ALL`, `name`, a linear `bound`), counted at each cost site: one per probe, one per ancestor probed, one per bucket or range element compared. The ratio test gains a measured clean bookkeeping in `crates/ddc-loadgen/src/verify.rs`: `plan_listed`, a synthesised `ExecReport` (every tenth item `NotRun`) with `fresh` a clone of the scan report with one rule changed (`refresh` returns at once when `fresh` is `None`, `after_clean.rs:306`, so this is what makes it and the `trash_on` candidate path run), `CleanReport::new` and `after_clean::edit` on a clone of the report; `tests/profiles.rs` checks it like `preflight`. The failing ratio lines are the "before".
3. `Edit` (`report/after_clean.rs`), built once in a shared `Edit::new` (`edit_tool_cleaned` builds one too): a `Path` index for `finding` (first bucket element not in `removed`) and `leave_entry` (probe `path.ancestors().skip(1)`, visit every element; it does not filter `removed`); the findings sorted by `Path` for `at_or_under` (`partition_point`, `take_while(starts_with)`, filter `removed`, hits sorted ascending); trash candidates for `trash_on`: a sorted set of finding indices, `refresh` inserting one when its new rule is `trash`, and each candidate re-checked for `rule == trash` and `dev` on the live finding when probed, so the lowest matching index wins as `position` does today; a lazy first-wins index of the fresh report for `refresh`; a set of `lines` probed by ancestors for `finish`. Paths are stable across `refresh` (its new finding was found by `==`), so the indexes stay valid.
4. `CleanReport::new` (`exec/report.rs`): `Path` maps of the refused and the run items, first insertion wins; paths are unique after `validate`.
5. `plan_listed` and `entry()` (`plan.rs`): the byte index with the same four-way match, and the KEPT set probed by ancestors. Each line is replaced in place, so the order of the refusal checks is unchanged.
6. The preflight's trashed check (`exec/preflight.rs`, its would-trash list built up front) and the executor's (`exec/mod.rs`, each path inserted as its entry is trashed): a set of the trashed paths probed with `ancestors().skip(1)`. The two are built differently and stay so.
7. `slopes` judges `clean-items`' user-time fit from one run per size: a per-command run count beside `JUDGED`, which stays the list of required commands with CPU and memory judged. The gate on `runs >= CPU_RUNS` (`scaling.rs:143`) becomes per command, and the test at `scaling.rs:372-374`, which asserts a single-run quadratic table is not judged, changes for `clean-items` and keeps its assertion for `scan`. The missing-command check (`scaling.rs:113`) and the one-size bail (`:126`) start applying to `clean-items`. Tests: a quadratic `clean-items` at one run per size fails, a linear one passes.
8. On a throwaway worktree, restore `leave_entry`'s scan with its comparison counted; record the failing line. Then the comparison of AC 2 and `make scaling` L `mixed` in enforce. If the fit is still over 1.2, profile at 2×M or L before changing anything else: the gap in the plan's introduction may be a site this plan does not index.
9. Runbook step 8: the out-of-scope paragraph becomes what was fixed; the guard and the journal go to the not-counted table (below).

Not in this story: `exec/guard.rs` `in_use` and `holding` check each item against every path and open file of the user's processes, linear in the tree with a constant set by the machine (63% of a dry run at M); `exec/journal.rs` syncs one line per item, the journal's crash record (ADR "Cleanup safety model: risk classes, plan and confirmation"). The guard's index (ancestor → first process, all holders in order for `holding`) is a follow-up story.

Tests that pin each refusal reason today (to stay green unchanged): `crates/ddc-core/tests/clean_items.rs` (`kept`, `rule-mismatch`, `changed-since-scan`, `action-not-allowed`), `recheck.rs` (`kept` against `changed-since-scan`), `user_data.rs` (`EntryLocked`, `Kept`, `ActionNotAllowed`, `Symlink`, `NotAnEntry`, `ParentChanged`, `OutsideScanRoots`, `inside-another-item`), `after_clean.rs` (refresh, a picked folder), `crates/ddc-cli/tests/clean_items_cli.rs`. None yet for the two cases of step 1, `trash_on` on two volumes, or a finding nested in two entry folders.

The harness's `clean-items` run accepts every item (`run.rs:301-310`), so `refresh`, `trash_on` after a rule changed by `refresh`, and the refused map of `CleanReport::new` run only in step 2's synthesised bookkeeping: that ratio test is their only guard against a quadratic. `edit_tool_cleaned` (`after_clean.rs:148`) builds the full index for a handful of `brew cleanup` items: O(findings) per call, which is what it costs today.

## Acceptance Criteria

- [ ] `make scaling SCALING_SIZE=L SCALING_PROFILE=mixed SCALING_LIMITS=enforce` judges `clean-items`' user-time fit from its one run per size and passes with it ≤ 1.2 (1.82, local slope 1.95, on `2af7940`), recorded here with the medians; the same tables before the fix fail it.
- [ ] The clean does the same: `golden_cli`, `load_cli` and the profile tests pass unchanged, `make load` reports the same items trashed, deleted and skipped per profile as before, and a `clean --items --json` of the L `mixed` items, run by the build before and the build after on two trees generated from the same seed, gives the same outcome and refusal reason per item (the two outputs compared, the diff recorded here as empty). A `--dry-run` would not do: the preflight has its own checks and never calls `plan_listed`, `Edit` or `CleanReport::new`. The planted items are all accepted (29 828 listed, none refused, at 10⁶), so the compared list adds refused ones: an item under a kept ancestor, an item removed between scan and clean, an item inside another listed folder, an item whose rule no longer matches; and the tests that assert each refusal reason are named here and pass unchanged.
- [ ] The ratio test measures `plan_listed` and the after-clean bookkeeping per profile, and fails on a restored per-item scan of the findings with its comparison counted (output recorded here).

## Final Summary

## Technical Notes

- The measurement came from the control run of the `aa64c83` step-1 regression (story "Complexity guard: operation counts and scaling exponents", Technical Notes). The `clean-items` system time grew with slope 1.03 on the same run, so the file moves are linear; the quadratic part is user time.
- `clean-items` is one run per size, not three: the harness trashes the items, so it cannot repeat on the same tree.

- Profile at M, 2026-10-29, `e933b7b`: `mixed` at 10⁵ entries (about 3 000 listed items), a busy machine (1-minute load 14), samply 0.13.1 at 1 kHz on the `profiling` build. Wrapped as `LOAD_CLI` so samply recorded only `clean --items` on the disk image of `scripts/load-image.sh` (runbook step 3 records at S only); samply's own output on stdout made the harness count the run as failed, which does not touch the recording. Frames resolved with `atos -i`; shares are of the main thread's CPU time.
  - `clean --items`: `exec::execute` 62.5%. `report::after_clean` `Edit::remove` 32.5%: `leave_entry`'s `findings.iter_mut().find` 21.0% and `at_or_under`'s filter over every finding 10.0%. `exec::delete::remove` 29.2%, of which `FileJournal::record` 17.6% (one `sync_data` per item: linear, the journal's durability, kept). `Guard::check` 6.8%.
  - `clean --items --dry-run`: `preflight` 98%. `Guard::check` 63.1%: `Guard::in_use` 43.1%, of which `Guard::holding` 24.5% and `guard::in_use`'s `paths.iter().any` 19.2%, each item against every path and open file of the user's processes. That is items × the processes' files: linear in the tree, with a constant set by the machine (many open files on a busy one), so not this story's quadratic, but the largest cost of a dry run. `preflight::walk_item` 26.8%.
  - `plan_listed` and the preflight's `trashed.iter().any` are not among the top frames at M. The quadratic part at M is `leave_entry` and `at_or_under`, about a third of a clean; at L, with ten times the items and the findings, it is the bulk of the 409 s.

## Dependencies

- Story "Complexity guard: operation counts and scaling exponents" (epic DDC-005): the counters, the ratio test and `make scaling` with its user-time fit.
- ADR "Cleanup safety model: risk classes, plan and confirmation": the listed-item match and the refusal reasons this story must not change.

## Attachments

-

---

*Last updated: 2026-10-29*

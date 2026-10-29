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
code_refs: ["crates/ddc-core/src/plan.rs", "crates/ddc-core/src/report/after_clean.rs", "crates/ddc-core/src/exec/preflight.rs", "crates/ddc-core/src/stats.rs", "crates/ddc-loadgen/src/scaling.rs"]
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
- `report/after_clean.rs`: `finding`, `at_or_under`, `trash_on`, `leave_entry`, `refresh`'s `fresh.findings.iter().find`, `remove_entry`'s `contents.entries.iter().position` and `finish`'s `lines.iter().any`, each called per item.

Which of them carry the 409 s is measured first, not assumed.

## Decomposition

- [ ] Profile `clean --items` on an L `mixed` tree (`make load-image`, then samply as in step 3 of the runbook "Profiling the core"), and record the top frames of user time here.
- [ ] Index the findings by path once per plan and once per after-clean report: a `HashMap<&OsStr, Vec<usize>>` keyed by `path.as_os_str()`, in finding order (`plan_listed` compares paths byte for byte, while `Path`'s own equality is by component, `/a/b/` equal to `/a/b`), since `Edit::finding` returns the first not-removed finding at a path and a report may hold several at one path. Answer the `KEPT` ancestor check with the `AtOrAbove` set from `rules/classify.rs`, and the preflight's trashed-ancestor check with a set of the would-trash paths probed once per ancestor (O(depth)). `at_or_under` becomes a range of the findings sorted by `Path`'s own order, which compares by component (`/a/b/x` before `/a/b-old`); a byte-wise sort would break the range. `trash_on` is per volume, a map of a few entries, not the findings index. The other lookups follow from what the profile shows.
- [ ] Keep the cleanup safety model as it is (ADR "Cleanup safety model: risk classes, plan and confirmation"): a listed item still matches on path, `(dev, ino)`, rule, class and action; the refusal reasons (`kept`, `changed-since-scan`, the others) and the order `entry()` checks them in are unchanged. Only how a finding is found changes.
- [ ] Count every per-item lookup in a new counter, `item_lookups` (one per index probe or per comparison), so that one counter keeps one kind of cost, and extend the in-process ratio test in `crates/ddc-loadgen/tests/profiles.rs` to measure `plan_listed` and the after-clean bookkeeping over the item list it already runs `preflight` on.
- [ ] Prove the test can fail: restore one per-item scan on a throwaway worktree, never committed, with its comparison counted as step 4 of "Adding a counter" in the runbook requires, and record the failing line. A scan restored verbatim, uncounted, passes the counters (the parent story measured this on `d9ad787`); the user-time fit of `make scaling` is the guard for that case.
- [ ] Judge `clean-items` in `ddc-loadgen slopes`: the run count becomes per command (`clean-items` runs once per size, since it trashes its items), and its user-time fit is judged like `scan`'s once it is linear. Its user time is seconds at n, far above the floor, and a single-run spread of ε moves the fit by about 0.65ε, so 10% costs about 0.07. Without this nothing automated guards it after the fix: the counters miss an uncounted loop.
- [ ] Replace the out-of-scope paragraph of step 8 in the runbook with what was fixed.

## Implementation Plan

## Acceptance Criteria

- [ ] `make scaling SCALING_SIZE=L SCALING_PROFILE=mixed SCALING_LIMITS=enforce` judges `clean-items`' user-time fit from its one run per size and passes with it ≤ 1.2 (1.82, local slope 1.95, on `2af7940`), recorded here with the medians; the same tables before the fix fail it.
- [ ] The clean does the same: `golden_cli`, `load_cli` and the profile tests pass unchanged, `make load` reports the same items trashed, deleted and skipped per profile as before, and a `clean --items --dry-run --json` of the L `mixed` items gives the same refusal reason per item before and after (the two outputs compared, the diff recorded here as empty). The planted items are all accepted (29 828 listed, none refused, at 10⁶), so the compared list adds refused ones: an item under a kept ancestor, an item removed between scan and clean, an item inside another listed folder, an item whose rule no longer matches; and the tests that assert each refusal reason are named here and pass unchanged.
- [ ] The ratio test measures `plan_listed` and the after-clean bookkeeping per profile, and fails on a restored per-item scan of the findings with its comparison counted (output recorded here).

## Final Summary

## Technical Notes

- The measurement came from the control run of the `aa64c83` step-1 regression (story "Complexity guard: operation counts and scaling exponents", Technical Notes). The `clean-items` system time grew with slope 1.03 on the same run, so the file moves are linear; the quadratic part is user time.
- `clean-items` is one run per size, not three: the harness trashes the items, so it cannot repeat on the same tree.

## Dependencies

- Story "Complexity guard: operation counts and scaling exponents" (epic DDC-005): the counters, the ratio test and `make scaling` with its user-time fit.
- ADR "Cleanup safety model: risk classes, plan and confirmation": the listed-item match and the refusal reasons this story must not change.

## Attachments

-

---

*Last updated: 2026-10-29*

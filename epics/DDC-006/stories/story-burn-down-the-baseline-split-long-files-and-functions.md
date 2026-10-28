---
type: story
id: "story-burn-down-the-baseline-split-long-files-and-functions"
epic: "DDC-006"
title: "Burn down the baseline: split long files and functions"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [rust, refactor, conventions]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Burn down the baseline: split long files and functions

| Field | Value |
|---|---|
| **Epic** | [DDC-006](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The checker's baseline records the code as it is. The epic is done only when the baseline is empty: every file, function and type within the limits, and every module within its allowed directions.

Measured on 2026-10-28 against the proposed limits:
- 3 files over 400 code lines: `rules/catalog.rs` 602, `scan/walker.rs` 573, `exec/mod.rs` 408.
- 11 functions over 70 lines, including:
  - `disk/mod.rs` 160;
  - `scan/walker.rs` 156 and 112;
  - `cmd/clean_items.rs` 131;
  - `cmd/clean.rs` 123;
  - `render.rs` 122 and 102.
- 7 two-module cycles in `ddc-core`.

The ADR's measured list and the baseline are authoritative; these numbers are only the starting estimate.

Every change here preserves behaviour. Each commit is a refactor only:
- the test suite passes;
- `make golden-check` (DDC-005 story "Synthetic load trees: large, wide, deep and hostile") passes on all S profiles;
- DDC-005's complexity counters do not grow, and its load comparison flags nothing.

## Decomposition

- [ ] Order by risk: the CLI and report files first, the safety path (`exec/*`, `plan.rs`, `rules/classify.rs`) last, after the low-risk files have proven the routine.
- [ ] Files: split along the cohesion components and the largest items `ddc-shape` reports. The baseline only shrinks.
- [ ] `rules/catalog.rs`: DDC-007's story "Rule catalog split: shared and per-OS sections" splits it by platform. Split further by category if a section is still over the limit, or apply the ADR's data-table rule.
- [ ] Long functions: extract named steps. Never wrap the same code in a closure to dodge the counter. The reviewer checks for this.
- [ ] Cycles: one commit per cycle. Start with the two cycles outside the safety path, `store↔disk` and `sys↔scan`, as they stand after DDC-007's move (move the shared types into `model` or a lower module). `exec↔plan↔report` comes last, after DDC-007's move behind the platform module, because it is a design change on the safety path, not a file split. Once the cycles are gone, the layer rules switch from warn to fail.
- [ ] Before and after each batch, run `make golden-check` (DDC-005 story "Synthetic load trees: large, wide, deep and hostile") on all S profiles.
- [ ] When the baseline is empty, delete it and switch the gate to "no baseline allowed".

## Implementation Plan

## Acceptance Criteria

- [ ] `shape-baseline.json` is gone; `make gate` passes with the hard limits, layer rules failing, and no baseline.
- [ ] Every exception in `shape.toml` has a reason and an owner; the number of exceptions is listed in this story.
- [ ] `make golden-check` passes on all S profiles against the outputs recorded before the first refactor commit.
- [ ] No operation counter of DDC-005's complexity guard grows at S/4·S, and `bench/compare_load.py` flags nothing against the pre-burn-down `bench/load-<date>.tsv` (its 15% rule).
- [ ] `projectstore:reviewer` confirmed for each batch that behaviour was preserved and no counter was gamed.

## Final Summary

## Technical Notes

- DDC-007's story "Move OS calls behind the platform module" moves `sys/`, `scan/walker.rs`, `exec/*` and `disk/*`. It lands first, or the work is done twice. Never run both at once.

## Dependencies

- Story "ddc-shape: a syn-based checker for size, nesting, cohesion and boundaries".
- DDC-005 stories "Coverage floor and property tests for the safety invariants" and "Synthetic load trees: large, wide, deep and hostile" (the safety net and the golden outputs).
- DDC-007 stories "Move OS calls behind the platform module" and "Rule catalog split: shared and per-OS sections" (ordering).

## Attachments

-

---

*Last updated: 2026-10-28*

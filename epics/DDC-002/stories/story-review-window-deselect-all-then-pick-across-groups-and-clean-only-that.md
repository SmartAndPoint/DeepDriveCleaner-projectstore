---
type: story
id: "story-review-window-deselect-all-then-pick-across-groups-and-clean-only-that"
epic: "DDC-002"
title: "Review window: Deselect All, then pick across groups and clean only that"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-30
updated: 2026-10-30
external_refs: {}
tags: [app, review-window, selection]
code_refs: ["app/Packages/DDCKit/Sources/DDCKit/ReviewModel.swift", "app/DeepDriveCleaner/ReviewView.swift", "app/Packages/DDCKit/Tests/DDCKitTests/DeselectAllTests.swift"]
specs: []
started_at: "2026-10-30T14:00:20Z"
closed_at: "2026-10-30T14:59:34.945Z"
plan_updated_at: null
---

# Review window: Deselect All, then pick across groups and clean only that

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | done |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner's request (2026-10-30): one button that clears the whole selection, so that they can start from nothing, tick a few rows in each group, and clean exactly those with the footer's button.

Today the Review window opens with the default selection: every selectable row of a class the scan offers by default. To keep only a few rows, the owner has to untick each group's Select all one by one. The footer's "Clean N items…" already takes the whole selection across every group, and a group's "Clean N from …" takes only that group's. What is missing is the way back to an empty selection.

The change:
- The footer gets **Deselect All** right after the summary it acts on, before the spacer ("N items selected · size"), away from Close and Clean. It clears every row's selection in every group, including picked entries of the user's own folders and rows hidden by the filter: after it, "0 items selected" is true.
- It is enabled whenever something is selected and the confirmation sheet is not open. A clean or a rescan does not disable it: the row checkboxes and a group's Select all stay live then too, and the reload after a clean only removes rows from the selection, so it cannot undo an empty one.
- It changes nothing else: the open group, the filter, the collapsed subgroups, the view (Groups or Items) and the rows stay as they are.
- "Clean N items…" is unchanged: after Deselect All and a few ticks, it opens the confirmation sheet holding only those rows. It already takes the whole selection, filter-hidden rows included; after Deselect All every selected row is one the user ticked by hand.

How long the empty selection lasts, as the selection does today:
- Scan Again and a Settings change keep it: the reload keeps the selection by identity.
- Opening the window from the popover (Review…, a biggest find, Clean selected…) loads the report afresh with the default selection. That is the way back to the default, and also a trap: hand-picked rows are lost by a round trip through the popover. Keeping the selection across that reopen is a separate decision, left out here.

Out of scope, deferred: an Edit menu item and a shortcut (Shift-Command-A, on the pattern of Find), and range selection (Shift-click) inside a group. A tri-state checkbox on the footer's summary was considered; a button names the action plainly.

## Decomposition

- [x] `ReviewModel.deselectAll()` and `canDeselectAll` (`!selection.isEmpty && sheet == nil`) in DDCKit.
- [x] The footer's Deselect All button in `ReviewView.swift`.
- [x] Unit tests, in their own file `DeselectAllTests.swift` rather than `ReviewSelectionTests`.
- [x] Version bump and release, through a PR: PR #7 (eea7d3c), published as 0.9.0 (build 14) on 2026-10-30.

## Implementation Plan

1. DDCKit first, with its tests: `deselectAll()` empties `selection`; `canDeselectAll` is false with an empty selection and while the sheet is open.
2. The button in `SelectionFooter`, after the summary and before the `Spacer`.
3. `make ddckit-test`, `make app-test`, the unsigned `xcodebuild` build.

## Acceptance Criteria

- [x] Unit test: with rows selected in two groups, a picked user-data entry, and a row hidden by the filter, `deselectAll()` leaves the selection empty; `selectedRows` is empty and `canClean(scope: nil)` is false. — evidence: `DeselectAllTests` "it clears every group, a picked entry and a row the filter hides; nothing is left to clean"
- [x] Unit test: after `deselectAll()` and ticking one row in each of two groups, the confirmation sheet holds exactly those two rows. — evidence: `DeselectAllTests` "then one row in each of two groups: the sheet holds exactly those two"
- [x] Unit test: `canDeselectAll` is true with something selected, and false with nothing selected or while the confirmation sheet is open. — evidence: `DeselectAllTests`: the first test (true, then false once cleared) and "it is off behind the confirmation sheet"
- [x] Unit test: after `deselectAll()`, the open group, the collapsed subgroups, the filter, the view and the rows are unchanged. — evidence: `DeselectAllTests` "the open group, the collapsed subgroups, the filter, the view and the rows stay as they were"
- [x] Unit test: after `deselectAll()`, Scan Again keeps the selection empty. — evidence: `DeselectAllTests` "Scan Again keeps the empty selection"
- [x] `make ddckit-test` and `make app-test` pass; the unsigned `xcodebuild` build passes. — evidence: Gate run 41068463547 on db7a4dc (app job green, 385 DDCKit tests); the unsigned `xcodebuild` build exit 0 before PR #6; Gate run 41072595744 on eea7d3c green

## Final Summary

**What changed.** The Review window's footer has **Deselect All** after the selection summary. `ReviewModel.deselectAll()` empties the whole selection: every group, picked user-data entries and rows the filter hides. `canDeselectAll` is on while something is selected and the confirmation sheet is closed. Nothing else moves: the open group, the filter, the collapsed subgroups and the view stay. "Clean N items…" then holds only the rows ticked afterwards. PR #6 (db7a4dc), released in 0.9.0 (PR #7, eea7d3c).

**Why.** The owner asked on 2026-10-30 for one way back to an empty selection, to pick a few rows across groups and clean only those; before it, every group's Select all had to be unticked by hand.

**Tests.** Five DDCKit tests in `DeselectAllTests.swift`, one per behaviour above. The Gate's app job passed on db7a4dc (385 tests) and on the release commit; the unsigned `xcodebuild` build passed before the PR.

**Risks and follow-ups.** Reopening the window from the popover still restores the default selection, as the Description says; keeping a hand-picked selection across that reopen is a separate decision. An Edit menu item and a shortcut, and Shift-click ranges, stay deferred. The owner's check is in the runbook "Owner checks: manual acceptance across stories".

## Technical Notes

- The owner's check, moved to the owner-checks runbook at close (done 2026-10-30): Deselect All clears every group's checkboxes; ticking a few rows in two groups and "Clean 2 items…" lists only those two.
- `selection` is a `Set<Int>` of row ids in `ReviewModel`; `set(_:selected:)` already ignores locked rows, and clearing needs no such filter, since locked rows never enter the set.

## Dependencies

- Story "Review window: after a clean, only skips that can clear stay selected" (DDC-002): `reload(afterClean:)` only removes rows from the selection, which is why a clean need not disable Deselect All.
- Story "Review window: clean only the selection in this group" (DDC-002): the group-scoped clean this complements.

---

*Last updated: 2026-10-30*

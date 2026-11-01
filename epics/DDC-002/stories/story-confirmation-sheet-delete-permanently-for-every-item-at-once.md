---
type: story
id: "story-confirmation-sheet-delete-permanently-for-every-item-at-once"
epic: "DDC-002"
title: "Confirmation sheet: Delete permanently for every item at once"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [app, review-window, clean]
code_refs: ["app/DeepDriveCleaner/ConfirmSheet.swift", "app/Packages/DDCKit/Sources/DDCKit/Confirmation.swift", "app/Packages/DDCKit/Sources/DDCKit/ReviewModel.swift", "app/Packages/DDCKit/Tests/DDCKitTests/DeleteAllPermanentlyTests.swift"]
specs: []
started_at: "2026-11-01T16:40:41.316Z"
closed_at: "2026-11-01T16:40:41.430Z"
plan_updated_at: "2026-11-01T16:40:41.316Z"
---

# Confirmation sheet: Delete permanently for every item at once

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

A tester cleaning 14 Review items (Python virtual environments, a uv tool) wanted all of them deleted rather than moved to the Trash, and had to tick Delete permanently 14 times: the sheet has the checkbox per item only. The Settings switch "Delete Review items instead of moving them to the Trash" (Automation → When you clean) sets where the checkboxes start, but it is a standing preference in another window, and the tester did not find it.

The confirmation sheet gets one checkbox above the item list, **Delete all permanently**, shown when two or more items can be deleted permanently (`Confirmation.Item.canBePermanent`: a `review` item the Trash would take). It is on when all of them are permanent, off when none is, and mixed otherwise. Clicking it when off or mixed makes every such item permanent; clicking it when on makes none permanent. Items that cannot be permanent are never touched. The per-item checkboxes stay. The sheet-wide checkbox is a one-off for this sheet: it does not change the Settings preference.

Changing it runs the warnings check again (`checkWarnings()`, which stops a check in flight and resets every prediction). A Trash move and a permanent delete are predicted differently (open files count as in use for a delete, never for a Trash move, story "Release 0.2.1: the Review window in the Dock, and skip what a running process holds open"), and one click changes up to every item, so leaving the old "No warnings" line over cleared predictions would be wrong for many items at once. The per-item checkbox keeps today's behaviour (clears that item's prediction only).

**Consent stays per item.** The safety ADR refuses a global `--permanent` with `--items` ("consent is per item", amendment of 2026-10-18). This checkbox is not that flag: it sets each item's own `permanent`, the list the clean receives still carries `permanent` per item (`Confirmation.listed`), and the Settings preference already starts every such item permanent. It adds no capability the ADR has not accepted; it saves clicks.

The checkbox's help text says the consequence: "Removed at once instead of moved to the Trash; they cannot be put back. Settings → Automation decides how the sheet starts."

## Decomposition

- [x] `Confirmation.permanentState: CheckState` (`off`, `mixed`, `on`, over `canBePermanent` items only; the existing `CheckState` of `GroupLayout.swift`), `Confirmation.showsAllPermanent` (two or more `canBePermanent` items) and `Confirmation.setAllPermanent(_ on: Bool) -> Bool` (whether anything changed), in DDCKit
- [x] `ReviewModel.setAllPermanent(_ on: Bool)`: only at the `confirm` stage; when anything changed, `checkWarnings()`
- [x] `ConfirmSheet`: the existing `CheckBox` view (moved out of `ReviewView.swift` if needed) above the list when `showsAllPermanent`, label "Delete all permanently", action `model.setAllPermanent(confirmation.permanentState != .on)`, help text as in the Description
- [x] DDCKit tests; the app builds in the Gate's `app` job

## Implementation Plan

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [x] Test (DDCKit): a confirmation of 3 `review`/`trash` items (the fixture `scan-report-groups` plus one `ReviewFixture.finding`), 1 `review` item whose action is not `trash`, and 1 `safe` item: `permanentState` is `off`; `setAllPermanent(true)` returns true, makes exactly the 3 permanent and `permanentState` is `on`; the other 2 items' `permanent` and `effectiveAction` are unchanged; `togglePermanent` of one of the 3 then gives `mixed`; `setAllPermanent(true)` from `mixed` gives `on`; `setAllPermanent(false)` gives `off`; `setAllPermanent(false)` again returns false — evidence: `app/Packages/DDCKit/Tests/DDCKitTests/DeleteAllPermanentlyTests.swift` `confirmationState`, PR #68
- [x] Test (DDCKit): a confirmation with no item that `canBePermanent` reports `off`, `showsAllPermanent` false, and `setAllPermanent(true)` changes nothing; with exactly one such item `showsAllPermanent` is false; with two it is true — evidence: `app/Packages/DDCKit/Tests/DDCKitTests/DeleteAllPermanentlyTests.swift` `threshold`, PR #68
- [x] Test (DDCKit, `ReviewModel`): at the `confirm` stage `setAllPermanent(true)` starts a new warnings check (the fake runner records a second dry run, with `--permanent`-per-item lines for the 3 items) and the warnings state leaves `checked`; a second `setAllPermanent(true)` that changes nothing starts no check; outside the `confirm` stage (cleaning, result) it changes nothing; the Settings preference `DeleteReviewItemsPermanently` is unchanged after it — evidence: `app/Packages/DDCKit/Tests/DDCKitTests/DeleteAllPermanentlyTests.swift` `reviewModel`: the check runs again (with no embedded CLI it fails at once, which shows it ran), a no-op starts none, the cleaning stage changes nothing, the preference is unchanged. The dry-run list's per-item `delete` is the list `listed(checkedAgainst:)` builds (`CleanRun.swift:18`), asserted by `listedPerItem`; the `result` stage is the same `stage == .confirm` guard (`ReviewModel.swift`), not asserted separately. Accepted at close on the reviewer's report
- [x] Test (DDCKit): the clean request built from the confirmation after `setAllPermanent(true)` sends `delete` for the 3 items, as it does when each was ticked by hand — evidence: `app/Packages/DDCKit/Tests/DDCKitTests/DeleteAllPermanentlyTests.swift` `listedPerItem`
- [x] `ConfirmSheet` shows the checkbox only when `showsAllPermanent` — evidence: `app/DeepDriveCleaner/ConfirmSheet.swift` (the `if confirmation.showsAllPermanent` block), predicate tested in `threshold`
- [x] `make ddckit-test` passes and the Gate's `app` job builds the app — evidence: `make ddckit-test` 453 tests passed; PR #68 Gate green, `app` job built the app; released in 0.14.10
- [ ] Owner check (moved to runbook "Owner checks: manual acceptance across stories"): in a sheet with several Review items, the checkbox ticks all; one per-item untick shows it mixed; a click when mixed ticks all; a click when all are ticked unticks all; the title and button change from "Move N items to the Trash?" to "Clean N items?" once any item is permanent; the warnings line shows the check running after each click; VoiceOver reads "partly selected" when mixed

## Final Summary

**What changed.** The confirmation sheet shows a "Delete all permanently" checkbox above the item list once two items can be deleted permanently (`Confirmation.showsAllPermanent`). It reuses the Review window's `CheckBox` with its mixed state (`Confirmation.permanentState: CheckState`); a click calls `ReviewModel.setAllPermanent(state != .on)`, which sets each item's own `permanent` (`Confirmation.setAllPermanent`) and, when anything changed, runs the warnings check again. The help text states the consequence and points at Settings. The Settings preference is untouched; the per-item checkboxes stay.

**Why.** A tester ticked Delete permanently 14 times (2026-11-01). Consent stays per item: the clean receives the same list as ticking each item by hand, so the safety ADR's refusal of a global `--permanent` with `--items` is not touched.

**Tests.** `DeleteAllPermanentlyTests` (4 tests); `make ddckit-test` 453 passed; PR #68 Gate green; released in 0.14.10. Reviewer: no blockers; one VoiceOver nit fixed in the PR (the label read once).

**Risks and follow-ups.** AC 3's dry-run-list and `result`-stage clauses are covered indirectly (see its evidence). The owner check is in the runbook.

## Technical Notes

- `Confirmation.items` is `private(set)`; the new mutation lives on the struct, as `togglePermanent` does.
- The app already has a mixed-state checkbox, `CheckBox` (`app/DeepDriveCleaner/ReviewView.swift`), with one action closure and an accessibility value for the mixed state; it is the Review window's Select all. Reuse it rather than `Toggle(_:sources:isOn:)`, whose shape is one `Binding` per element and whose mixed-state click is not documented.

## Dependencies

- None. The preference `DeleteReviewItemsPermanently` came with story "Review and clean window"; its Settings switch with story "Settings: folders, types, notifications".

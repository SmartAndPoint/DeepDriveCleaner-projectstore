---
type: story
id: "story-review-window-clean-only-the-selection-in-this-group"
epic: "DDC-002"
title: "Review window: clean only the selection in this group"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-21
updated: 2026-10-31
external_refs: {}
tags: [app, review]
code_refs: ["README.md", "app/DeepDriveCleaner/ReviewGroupsView.swift", "app/DeepDriveCleaner/ReviewView.swift", "app/Packages/DDCKit/Sources/DDCKit/Formatting.swift", "app/Packages/DDCKit/Sources/DDCKit/ReviewModel.swift", "app/Packages/DDCKit/Tests/DDCKitTests/GroupCleanTests.swift"]
specs: []
started_at: "2026-10-21T11:04:18.237Z"
closed_at: "2026-10-31T10:30:46.980Z"
plan_updated_at: "2026-10-21T11:04:18.237Z"
---

# Review window: clean only the selection in this group

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | done |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The Review window's only clean action, the footer's "Clean N items…", takes the whole selection across every group. The owner wants to clean just what is selected in the group that is open, without unticking the other groups first (2026-10-21).

Artboard 12 of the design canvas ("Clean only this group") drew three placements. The owner chose **B**: the action sits at the right end of the group's Select all row, after the summary.
- The summary reads "2 of 3 · 7.2 GiB": what is selected among the rows the row acts on, and its size.
- The button reads "Clean 2 from App caches…".

It opens the same confirmation sheet as the footer's button, holding only those items. The footer keeps cleaning the whole selection.

## Decomposition

Rough until the plan gate.

- [x] DDCKit:
  - `ReviewModel.openSheet` takes an optional sidebar entry. The model derives the scope itself, after the `refreshSettings()` it already does, as `items.layout(entry, matching: query).selectAll`. Ids are indices into `report.findings`, and `reload()` remaps the selection, not ids a view captured earlier.
  - The sheet is `Confirmation(items, selection: selection ∩ scope)`.
  - Everything after that is unchanged: the dry-run warnings, per-item permanent delete, the clean and the result.
- [x] Scope:
  - **Rows.** The scope is the selectable rows the Select all row acts on, so with a filter typed it is the rows the filter shows, as Select all is.
  - **Why the filter counts.** The default selection ticks rows the user never looked at. With a filter typed, the button acts on what the user sees, as Select all does.
  - **Nested items.** The label counts the scoped selection exactly as the sheet holds it, with no covered-row subtraction. Only user-data folders cover rows, and that group has no Select all row. A `node_modules` under a picked `~/Documents/proj` is therefore sent on its own by its group's button, and counted once by the footer's button.
- [x] Wording: `Wording` builds the button's label ("Clean 1 from App caches…", "Clean 12 from Python environments…") and the summary's "N of M · size".
- [x] App: `SelectAllRow` (`app/DeepDriveCleaner/ReviewGroupsView.swift`) gains the bordered button.
  - **Enabled** only when the scope holds a selected row and no clean, rescan or sheet is running, as the footer's button is.
  - **Not shown** where there is no Select all row: the items mode, and the User data group, whose files are picked one by one.
- [x] After a scoped clean, the selection in other groups stays. `reload()` keeps the selection by identity across the rescan, and `make()` locks the cleaned identities (`ReviewModel.swift`).
- [x] The summary's bytes are the selected rows the filter shows. They can be smaller than the group header's amount, which counts the whole group regardless of the filter.
- [x] A long group name in the label truncates in the middle, and the full name stays in its help tag.
- [x] With nothing in the scope selectable, the row keeps its total ("9 folders · 3.8 GiB") and shows no button.

## Implementation Plan

Planner, 2026-10-21. It ships with "HIG pass: keyboard, colour, empty states and menus" and comes after it.

1. **DDCKit `ReviewModel`:**
   - `selectedRows(in scope:)`, which is `selection ∩ scope` less what `items.covered(by:)` covers inside that set. This is what `Confirmation(items, selection:)` builds, so the label, the summary and the sheet agree. The case the critic raised, a covered row whose folder lies outside the scope, therefore counts it in the label and sends it with the group.
   - `canClean(scope:)`, which needs a non-empty scoped selection with no sheet, rescan or clean running. The footer uses it too.
   - `openSheet(scope: [Int]? = nil)`. It captures the scope's identities before `refreshSettings()`, because a `reload()` can renumber rows, and maps them back after it.
2. **Wording.** `cleanFrom(_:group:)` gives "Clean 2 from App caches…"; `scopeSummary(selected:of:bytes:)` gives "2 of 3 · 7.2 GiB", or "0 of 3" with nothing selected.
3. **`GroupCleanTests.swift`:**
   - two groups, scoped and unscoped sheets;
   - a filter that hides a selected row;
   - the user-data nested case;
   - the selection kept after a scoped clean and Done;
   - the wording and `canClean`.
4. **App.** `SelectAllRow(model:ids:group:)` shows the summary and the bordered button, which calls `openSheet(scope: ids)`, has no shortcut, and is disabled by `canClean`.

No new API may be named `.clean(`: `cleanHasOneOwner` matches `/\.clean\b/`.

Check: `make app-test`, the unsigned build, the reviewer, then the manual check.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [x] Unit test: with rows selected in two groups, opening the sheet scoped to one group gives a confirmation holding only that group's selected items, with their count and bytes. The unscoped sheet still holds both groups. — evidence: `GroupCleanTests.scoped`
- [x] Unit test: with a filter that hides one of the group's selected rows, the scoped sheet leaves that row out. — evidence: `GroupCleanTests.filtered`
- [x] Unit test: the scope crosses the reload before the sheet by identity. A newer report with its rows renumbered opens the same items. — evidence: `GroupCleanTests.scopeByIdentity`
- [x] Unit test: with a user-data folder picked and a `node_modules` inside it selected in its own group, the scoped sheet of that group holds the `node_modules`, and the label's count equals the sheet's. — evidence: `GroupCleanTests.insidePickedFolder`
- [x] Unit test: after a scoped clean of App caches, the result's Done leaves the Python environments rows selected. — evidence: `GroupCleanTests.keepsTheRest`
- [x] Unit test: the label reads "Clean 2 from App caches…". The summary reads "2 of 3 · 7.2 GiB". With nothing selected in the scope, the button is disabled. — evidence: `GroupCleanTests.wordingAndEnabling` (the group is called "Browser and app caches" today, and the test uses that name)
- [x] `make app-test` passes. The unsigned `xcodebuild` build passes. — evidence: the work landed in af8c590 and shipped in 0.2.2, whose `make dist` then required `app-test` and builds the app with `xcodebuild … CODE_SIGNING_ALLOWED=NO` before signing (`scripts/build-app.sh`); there was no CI Gate yet. Today the Gate's `app` job runs `make app-test`, green on main at 2cdae98 (run 41191429282, 2026-10-31).
- [ ] Manual (owner, moved to the owner-checks runbook at close, 2026-10-31):
  - With App caches and Python environments selected, "Clean N from App caches…" opens a sheet listing only App caches. Cleaning it leaves the Python environments on disk.

## Final Summary

**What changed.** Each group's Select all row has a "Clean N from <group>…" button that opens the sheet on that group's selection only. Shipped in 0.2.2 (af8c590).
- `ReviewModel.openSheet` takes an optional sidebar entry and derives the scope itself, after the settings refresh, as the selectable rows Select all acts on (with a filter typed, the rows the filter shows). The sheet holds the selection within the scope; the dry run, the clean and the result are unchanged.
- `Wording.cleanFrom` and `Wording.scopeSummary` build the label and the "N of M · size" line. The button is not shown in the Items view or in User data.
- After a scoped clean the other groups' rows stay selected, by identity across the rescan.

**Why.** The owner (2026-10-21): the footer's Clean takes the whole selection across every group, and the owner wanted to clean just the open group's selection without unticking the others first. Placement B of artboard 12.

**Tests.** `GroupCleanTests` as above; the 0.2.2 release build; the Gate's `app` job on main today. Closed under the owner's rule of 2026-10-30: the manual check moved to the runbook "Owner checks: manual acceptance across stories".

**Risks and follow-ups.**
- The group named "App caches" here is "Browser and app caches" today; the runbook check uses the current name.

## Technical Notes

- **Where the scope comes from.** `SelectAllRow.ids` is already "every selectable row the filter shows" (`ReviewGroupsView.swift`), so the scope is the ids it has.
- **Keyboard.** The footer's button keeps `.defaultAction`. The new button has no shortcut.
- **The clean contract.** `ContractTests.cleanHasOneOwner` is untouched: it checks who runs `.clean`, `--items` and `cleanConfirmedItems(`, and any caller may open the sheet.

## Dependencies

- "Review window: sorted sidebar, Select all and folder groups" (DDC-002): the Select all row this extends.
- It ships with "HIG pass: keyboard, colour, empty states and menus" (DDC-002); both change `ReviewView.swift` and `ReviewGroupsView.swift`.

## Attachments

- Design canvas: https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV (artboard 12, variant B)

---

*Last updated: 2026-10-31*

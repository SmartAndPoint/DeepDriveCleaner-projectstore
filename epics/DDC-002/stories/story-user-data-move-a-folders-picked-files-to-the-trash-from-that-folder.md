---
type: story
id: "story-user-data-move-a-folders-picked-files-to-the-trash-from-that-folder"
epic: "DDC-002"
title: "User data: move a folder's picked files to the Trash from that folder"
status: in-progress
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-29
external_refs: {}
tags: [app, review, user-data, user-report]
code_refs: ["app/DeepDriveCleaner/ConfirmSheet.swift", "app/DeepDriveCleaner/ReviewGroupsView.swift", "app/Packages/DDCKit/Sources/DDCKit/Confirmation.swift", "app/Packages/DDCKit/Sources/DDCKit/Formatting.swift", "app/Packages/DDCKit/Sources/DDCKit/GroupLayout.swift", "app/Packages/DDCKit/Sources/DDCKit/ReviewModel.swift", "app/Packages/DDCKit/Sources/DDCKit/SkipSummary.swift", "app/Packages/DDCKit/Tests/DDCKitTests/PickedMoveTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/UserDataTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ContractTests.swift"]
specs: []
started_at: "2026-10-29T09:16:41.169Z"
closed_at: null
plan_updated_at: "2026-10-29T09:16:41.169Z"
---

# User data: move a folder's picked files to the Trash from that folder

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

Requested by the owner on 2026-10-28, after a user's screen recording (app 0.4.0): "in User data … there is no 'delete selected' button, though one is called for".

What the recording shows. In Movies the user ticked one entry (a 2.7 GiB folder). Nothing near it offered to move it. The only way was the footer's **Clean 11 items…**, which also held the ten items ticked elsewhere: caches, `node_modules`, the Docker item. Every group of the safe, rebuildable and review classes, the Trash aside, has a **Select all** row with a **Clean N from <group>…** button (story "Review window: clean only the selection in this group"). The User data groups have neither, by design: their layout (`GroupLayout.entriesLayout`) returns `selectAll: nil`, so no row is drawn.

Select all stays out. The safety ADR's amendment of 2026-10-20 names "no default selection, no Select all for user data" as the mitigation for sending a user's own files to the Trash, and this story does not touch it. What the folder lacks is not a way to pick many, but a way to act on what was picked here, one by one, without the rest of the window's selection. So:

- A User data folder whose entries can be picked (the `ENTRY_RULES` of `rules/entries.rs`: Downloads, Pictures, Movies, Music and device backups) gets a header row with **no checkbox**: "2 picked · 3.1 GiB" (the picked findings' `bytes`, as the Select all row and the sheet count them, not the group header's own bytes) and a **Move 2 from Movies to the Trash…** button. With nothing picked, it says "Pick files below to move them to the Trash" and the button reads "Move to the Trash…", disabled. The scope is the entries shown: a typed filter narrows it, as it narrows Select all elsewhere, so a picked entry the filter hides is neither counted nor moved. When the filter hides every pickable entry, the scope is empty and no row is drawn, as Select all does. A group has either a Select all row or this row, never both: `pickScope` is empty for every shape but the entries one.
- The button opens the same confirmation sheet as the footer, scoped to this folder's picked entries (`openSheet(scope:)` with the folder's ids), as the other groups' buttons do. The sheet's "Your own files" block comes first, as today.
- Device backups get the button too, as "Move 1 from iPhone and iPad backups to the Trash…" (the group's name, `CleanWording.userDataName`): the ADR's amendment accepted that one run can send whole backups to the Trash, and the sheet's "Your own files" block, which comes first, names each backup with its size.
- The verb is "Move … to the Trash", never "Clean" or "Delete": these entries only go to the Trash. The sheet it opens follows: when every confirmed item goes to the Trash, its title and button say "Move 2 items to the Trash?" and "Move to the Trash" instead of "Clean 2 items". The rule is computed over each item's effective action (`trashCount`), never the row's default. It covers the confirm stage's title and button only: the cleaning stage, the footer's "Clean N items…" and the header's "frees about" stay as they are. The footer's mixed sheet keeps "Clean". The rule is the sheet's, not User data's: any sheet whose every item goes to the Trash says "Move", a Review-only one included, and turning on Delete permanently for an item turns it back to "Clean" at once.
- User data groups with nothing to pick (Telegram, WhatsApp, Cloud folders: report only, no entries) get no button.

- A finding selected on its own inside a picked Downloads folder (a `node_modules`) lies outside the scope, which holds entry ids only: the scoped sheet does not say "Includes N selected on their own", and the finding goes to the Trash with the folder, as the footer's sheet would send it. Accepted; the footer's sheet still reports it.

Nothing in the CLI changes: `clean --items` already takes picked entries, and the scoped sheet already exists.

## Decomposition

- [x] DDCKit: `GroupLayout` for the entries shape gains `pickScope: [Int]` (the entries shown that can be picked), separate from `selectAll`, which stays nil for user data. `Wording.moveFrom(count, group:)` beside `Wording.cleanFrom` in `Formatting.swift`, and `Wording.picked(count:bytes:)`.
- [x] DDCKit: the sheet's title and button wording when every confirmed item's action is trash (`Confirmation.trashCount == items.count`).
- [x] App: `GroupView` draws a `PickedRow` (no checkbox, the count and the button) when `pickScope` is non-empty; the button calls `model.openSheet(scope: pickScope)` and is disabled by `!model.canClean(scope:)`.

## Implementation Plan

From the planner's pass of 2026-10-29, adjusted by the critic's. DDCKit and the app only.

1. `GroupLayout.swift`: `pickScope: [Int]` (default empty, so the other construction sites stay), set in `entriesLayout` to the shown items that are selectable; `selectAll` stays nil.
2. `Formatting.swift`: `Wording.moveFrom(_:group:)` beside `cleanFrom`, and `Wording.picked(count:bytes:)`. `ReviewModel.swift`: `pickedSummary(_:)` from `selectedRows(in:)`, summing `finding.bytes`.
3. `Confirmation.swift`: `movesOnlyToTrash` over `trashCount`. `SkipSummary.swift` (the `CleanWording` extension with `sheetHeader`): `sheetTitle(_:)` and `sheetAction(_:)`. `CleanWording.swift` is left alone (the conda story edits it).
4. `ConfirmSheet.swift`: the title and the Clean button take those functions; `ContractTests.cleanHasOneOwner` matches by the call in the same change.
5. `ReviewGroupsView.swift`: `PickedRow` beside `SelectAllRow`, no checkbox, its button calls `model.openSheet(scope: ids)`, disabled by `!model.canClean(scope:)`; drawn in `GroupView` when `selectAll` is nil and `pickScope` is not empty.
6. Tests: `UserDataTests.swift` (a Movies, Telegram and WhatsApp fixture behind a flag, so existing ids stay), `GroupCleanTests.swift`, `ReviewGroupsTests.swift`, `ContractTests.swift` (the PickedRow contract).
7. `make gate`, then `projectstore:reviewer`.

## Acceptance Criteria

- [x] Test (DDCKit): the Movies layout has `selectAll == nil` and a `pickScope` of its pickable entries; locked entries (an app's library, "iMovie manages it") are not in it. — evidence: `PickedMoveTests.scope` (app/Packages/DDCKit/Tests/DDCKitTests/PickedMoveTests.swift)
- [x] Test (DDCKit): a typed filter narrows `pickScope` to the entries it shows, and to empty when it hides them all; every non-entries layout has an empty `pickScope`. — evidence: `PickedMoveTests.filtered`
- [x] Test (DDCKit): a Downloads folder entry picked with a `node_modules` inside it selected on its own: the scoped sheet holds the entry only and says nothing of the `node_modules`. — evidence: `PickedMoveTests.insideAPickedFolder`
- [x] Test (DDCKit): with two Movies entries and one cache picked, the scoped sheet's confirmation holds the two entries only, both to the Trash; after the scoped Move and Done (a fake CLI), the cache is still selected. — evidence: `PickedMoveTests.sheet` (two entries and a cache), `PickedMoveTests.keepsTheRest` (Move and Done with a fake CLI)
- [x] Test (DDCKit): the button's label is "Move 2 from Movies to the Trash…" and the header reads "2 picked · <size>"; with nothing picked the header reads "Pick files below to move them to the Trash", the label "Move to the Trash…", and the scope cannot be cleaned. — evidence: `PickedMoveTests.wording`
- [x] Test (DDCKit): Telegram and WhatsApp layouts have no `pickScope`; a sheet of trash-only items is titled "Move 1 item to the Trash?" with the button "Move to the Trash", a mixed one "Clean 2 items?" and "Clean 2 items"; a Review-only trash sheet says "Move", and turning on Delete permanently for its item makes it "Clean". — evidence: `PickedMoveTests.reportOnly`, `PickedMoveTests.sheetWording`
- [x] Contract test (source check): `SelectAllRow(` occurs once in `ReviewGroupsView.swift`, inside `if let all = layout.selectAll`; the `PickedRow` body contains no `CheckBox(`, `toggleAll(`, `set(` or `selection`, and its one button calls only `model.openSheet(scope: ids)`. `ContractTests.cleanHasOneOwner` matches the sheet's button by its call, `/Button\([^\n]*\{ model\.cleanConfirmedItems\(\) \}\n(?:\s*\.[^\n]*\n)*/`, since the label is now computed; its `role: .destructive` and `.disabled(… !sheet.cleanAllowed)` assertions and the one-owner count stay. — evidence: `ContractTests.userDataPickedRow`, `ContractTests.cleanHasOneOwner`
- [x] `make app-test` passes and the unsigned app builds — evidence: `swift test` 294 tests passed; `xcodebuild … CODE_SIGNING_ALLOWED=NO build` succeeded (`make app` signs, `make gate` does not build the app)
- [ ] Manual (owner): in Movies, pick one entry, press the button: the sheet holds that entry only and moves it to the Trash.

## Final Summary

## Dependencies

- "User data: pick your own files for the Trash" (DDC-002): the entries and their picking
- "Review window: clean only the selection in this group" (DDC-002): the scoped sheet
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model]], amendment of 2026-10-20: no Select all for user data, kept
- "Review window: Clean is not offered when every item would be skipped" (DDC-002): the other stories from the same recording

## Attachments

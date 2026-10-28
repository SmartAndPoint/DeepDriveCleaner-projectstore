---
type: story
id: "story-user-data-move-a-folders-picked-files-to-the-trash-from-that-folder"
epic: "DDC-002"
title: "User data: move a folder's picked files to the Trash from that folder"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [app, review, user-data, user-report]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# User data: move a folder's picked files to the Trash from that folder

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

Requested by the owner on 2026-10-28, after a user's screen recording (app 0.4.0): "in User data … there is no 'delete selected' button, though one is called for".

What the recording shows. In Movies the user ticked one entry (a 2.7 GiB folder). Nothing near it offered to move it. The only way was the footer's **Clean 11 items…**, which also held the ten items ticked elsewhere: caches, `node_modules`, the Docker item. Every group of the safe, rebuildable and review classes has a **Select all** row with a **Clean N from <group>…** button (story "Review window: clean only the selection in this group"). The User data groups have neither, by design: their layout (`GroupLayout.entriesLayout`) returns `selectAll: nil`, so no row is drawn.

Select all stays out. The safety ADR's amendment of 2026-10-20 names "no default selection, no Select all for user data" as the mitigation for sending a user's own files to the Trash, and this story does not touch it. What the folder lacks is not a way to pick many, but a way to act on what was picked here, one by one, without the rest of the window's selection. So:

- A User data folder whose entries can be picked (the `ENTRY_RULES` of `rules/entries.rs`: Downloads, Pictures, Movies, Music and device backups) gets a header row with **no checkbox**: "2 picked · 3.1 GiB" and a **Move 2 from Movies to the Trash…** button. With nothing picked, it says "Pick files below to move them to the Trash" and the button is disabled.
- The button opens the same confirmation sheet as the footer, scoped to this folder's picked entries (`openSheet(scope:)` with the folder's ids), as the other groups' buttons do. The sheet's "Your own files" block comes first, as today.
- Device backups get the button too, as "Move 1 from iPhone backups to the Trash…": the ADR's amendment accepted that one run can send whole backups to the Trash, and the sheet's "Your own files" block, which comes first, names each backup with its size.
- The verb is "Move … to the Trash", never "Clean" or "Delete": these entries only go to the Trash. The sheet it opens follows: when every confirmed item goes to the Trash, its title and button say "Move 2 items to the Trash?" and "Move to the Trash" instead of "Clean 2 items". The footer's mixed sheet keeps "Clean".
- User data groups with nothing to pick (Telegram, WhatsApp, Cloud folders: report only, no entries) get no button.

Nothing in the CLI changes: `clean --items` already takes picked entries, and the scoped sheet already exists.

## Decomposition

- [ ] DDCKit: `GroupLayout` for the entries shape gains `pickScope: [Int]` (the entries shown that can be picked), separate from `selectAll`, which stays nil for user data. `CleanWording.moveFrom(count, group:)`.
- [ ] DDCKit: the sheet's title and button wording when every confirmed item's action is trash (`Confirmation.trashCount == items.count`).
- [ ] App: `GroupView` draws a `PickedRow` (no checkbox, the count and the button) when `pickScope` is non-empty; the button calls `model.openSheet(scope: pickScope)` and is disabled by `!model.canClean(scope:)`.

## Implementation Plan

## Acceptance Criteria

- [ ] Test (DDCKit): the Movies layout has `selectAll == nil` and a `pickScope` of its pickable entries; locked entries (an app's library, "iMovie manages it") are not in it.
- [ ] Test (DDCKit): with two Movies entries and one cache picked, the scoped sheet's confirmation holds the two entries only, both to the Trash, and the cache stays selected after the sheet closes.
- [ ] Test (DDCKit): the button's label is "Move 2 from Movies to the Trash…", and with nothing picked the scope cannot be cleaned.
- [ ] Test (DDCKit): Telegram and WhatsApp layouts have no `pickScope`; a sheet of trash-only items is worded "Move … to the Trash", a mixed one "Clean".
- [ ] Contract test (source check): no User data view draws a "Select all" checkbox; the new button calls only `openSheet(scope:)`.
- [ ] Manual (owner): in Movies, pick one entry, press the button: the sheet holds that entry only and moves it to the Trash.

## Final Summary

## Dependencies

- "User data: pick your own files for the Trash" (DDC-002): the entries and their picking
- "Review window: clean only the selection in this group" (DDC-002): the scoped sheet
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model]], amendment of 2026-10-20: no Select all for user data, kept
- "Review window: Clean is not offered when every item would be skipped" (DDC-002): the other stories from the same recording

## Attachments

---
type: story
id: "story-the-trash-its-own-row-not-your-data"
epic: "DDC-002"
title: "The Trash: its own row, not Your data"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [core, app, review, trash, clean-result, user-report]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# The Trash: its own row, not Your data

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

From a user's screen recording of 2026-10-28 (app 0.4.0). The user cleaned Python environments, ML models, `node_modules` and a large file from Movies; all of them are moved to the Trash, not deleted. After each clean the Review sidebar showed a group called **Your data** growing: 8 KiB, then 5.9 GiB, 8.3 GiB, 10.9 GiB. Opened, it held one row, `~/.Trash`, "Manual only: ddc has no command for it · Empty the Trash in Finder". The result of each clean said "Free space 240.1 → 240.1 GiB" next to "25 to the Trash · 2.4 GiB". To the user, what they had just removed came back under a new name and freed nothing.

Two things are right and stay: a move to the Trash frees nothing until the Trash is emptied, and DeepDriveCleaner never empties it (safety ADR §1, the `none` method: the Trash is report only). What is wrong is how it is shown:

- **The name.** The `trash` rule sits in the rule listing's "Your data" group (`rules/listing.rs`), beside `xcode.archives`, `ide.workspace-storage` and the other report-only rules of the review class. The sidebar shows the group's label. The Trash gets its own group, **Trash**, with its own sidebar entry under Review. Its blurb says: "What was moved to the Trash, by you or by a clean. Its space comes back when you empty the Trash in Finder." The entry has an **Open the Trash** button that opens the Trash in Finder (`NSWorkspace.shared.open` on the home folder's `.Trash`, no AppleScript); there is no Empty button.
- **What a clean says.** When a clean moved items to the Trash, the result's free-space block says so instead of leaving 240.1 → 240.1 GiB to speak for itself: "2.4 GiB went to the Trash. It comes back when you empty the Trash." with the same Open the Trash button. "APFS may take a moment to give the space back" stays for deleted items only.
- **The totals.** The Trash's bytes are not in any "frees" number: not in the popover's reclaimable total, not in the footer's or the sheet's "frees about". Today that holds only because a `none` finding never enters the plan or the selection (`plan.rs`, `NotSelected`); a finding's own reclaimable bytes are `private_bytes` or `bytes`, with no exception for its method (`Finding::reclaimable_bytes`, Swift `reclaimableBytes`). The tests pin the totals on both sides, where a regression would show.
- **Settings → Types.** No new row. The Types pane lists a group only when it has a switchable rule (`TypesRows.make`; `switchable` is false for a rule with no action), and `trash` has none, as it has none in "Your data" today. A test pins that the "Trash" group gives no Types row.

The rule's class, method and id do not change, so the safety model is untouched. The group label is part of `ddc rules --json` (`docs/rules.schema.json` does not constrain labels), and the app takes the sidebar's labels and order from it.

## Decomposition

- [ ] Core: `rules/listing.rs` gains a "Trash" group holding `trash`, ordered after "Interrupted cleanups"; "Your data" loses it. The `rules --json` golden test and the DDCKit fixture `rules.json` are updated.
- [ ] DDCKit: the Trash group's blurb in `CleanWording`; `CleanResult` exposes the bytes moved to the Trash, and `CleanWording` the result sentence.
- [ ] App: the sidebar entry and its Open the Trash button; `CleanResultView` shows the Trash sentence and button when `trashed.count > 0`.

## Implementation Plan

## Acceptance Criteria

- [ ] Test (core): `ddc rules --json` lists `trash` in the "Trash" group and in no other.
- [ ] Test (core): a scan of a temp HOME with a `.Trash` of known size reports the `trash` finding as not selected, and `selected_reclaimable_bytes` leaves its bytes out.
- [ ] Test (DDCKit): `ReviewItems` puts the Trash finding in its own sidebar entry named "Trash" in the review section, and its row cannot be selected; `ReportSummary.reclaimable(including:)` leaves it out even with `.review` included.
- [ ] Test (DDCKit): `TypesRows.make` over the updated `rules.json` fixture gives no "Trash" row, and the other rows are unchanged.
- [ ] Test (DDCKit): a clean result with trashed items says how many bytes went to the Trash; one with deleted items only does not.
- [ ] Manual (owner): move something to the Trash with a clean, Scan Again: the Trash row has grown and is called Trash; Open the Trash opens Finder's Trash.

## Final Summary

## Dependencies

- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model]] §1: the Trash is report only; this story keeps it so
- "Review window: sorted sidebar, Select all and folder groups" (DDC-002): the sidebar entries
- "Review and clean window" (DDC-002): the result view
- "Review window: Clean is not offered when every item would be skipped" (DDC-002): the other stories from the same recording

## Attachments

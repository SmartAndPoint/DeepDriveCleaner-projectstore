---
type: story
id: "story-the-trash-its-own-row-not-your-data"
epic: "DDC-002"
title: "The Trash: its own row, not Your data"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-30
external_refs: {}
tags: [core, app, review, trash, clean-result, user-report]
code_refs: ["crates/ddc-core/src/rules/listing.rs", "crates/ddc-core/tests/rules.rs", "crates/ddc-core/tests/report.rs", "crates/ddc-cli/tests/rules_cli.rs", "bench/golden/rules.json", "app/Packages/DDCKit/Sources/DDCKit/ReviewItems.swift", "app/Packages/DDCKit/Sources/DDCKit/CleanWording.swift", "app/Packages/DDCKit/Sources/DDCKit/GroupLayout.swift", "app/Packages/DDCKit/Sources/DDCKit/AfterClean.swift", "app/Packages/DDCKit/Tests/DDCKitTests/TrashTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/SettingsTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/CleanResultTests.swift", "app/DeepDriveCleaner/OpenTrashButton.swift", "app/DeepDriveCleaner/CleanResultView.swift", "app/DeepDriveCleaner/ReviewGroupsView.swift"]
specs: []
started_at: "2026-10-29T05:12:09.708Z"
closed_at: "2026-10-30T06:45:43.235Z"
plan_updated_at: "2026-10-29T05:12:09.708Z"
---

# The Trash: its own row, not Your data

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
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

- [x] Core: `rules/listing.rs` gains a "Trash" group holding `trash`, ordered after "Interrupted cleanups"; "Your data" loses it. The `rules --json` golden test and the DDCKit fixture `rules.json` are updated.
- [x] DDCKit: the Trash group's blurb in `CleanWording`; `CleanResult` exposes the bytes moved to the Trash, and `CleanWording` the result sentence.
- [x] App: the sidebar entry and its Open the Trash button; `CleanResultView` shows the Trash sentence and button when `trashed.count > 0`.

## Implementation Plan

From the planner, 2026-10-29. The story needs no new module and no schema change.

1. **Core.** `rules/listing.rs` `GROUPS` gains `("Trash", &["trash"])` after "Interrupted cleanups", and "Your data" loses `trash`. Then:
   - `tests/rules.rs` allows "Trash" among the groups with no switch;
   - `rules_cli.rs` lists the new group and its one member;
   - a `report.rs` test scans a `.Trash` and pins `selected_reclaimable_bytes`.
2. **Regenerated files.**
   - `make golden-record` re-records the goldens; only `bench/golden/rules.json` changes.
   - The DDCKit `Fixtures/rules.json` is regenerated from `rules --json` under a temp HOME.
   - `docs/rules.schema.json` is unchanged: the schema does not list labels.
3. **DDCKit.** The sidebar takes its labels from `rules --json`, so the Swift side needs only these changes:
   - `trash` leaves `userDataOrder` and `CleanWording.userDataName`. Otherwise "Trash" would be in `groupOrder` twice, and the sidebar would show two identical entries.
   - `GroupLayout` finds the Trash by its rule id (`ReviewItems.trashRule`). It gives the group `CleanWording.trashBlurb`, no Select all, no repeated hint, and `opensTrash`.
   - `CleanWording.trashed(_:)` and `apfsNote(_:)` are added in `AfterClean.swift`.
4. **App.**
   - A new `OpenTrashButton` (`NSWorkspace.shared.open` on `~/.Trash`).
   - The Trash group's header shows the button.
   - `CleanResultView` shows one Trash line for the whole result, and the APFS note only through `apfsNote`.
5. **One decision beyond the story's text.** The APFS note shows when something was deleted **or a tool ran**: a tool command such as `go clean -cache` frees space now too. A clean that only moved items to the Trash, or only skipped them, shows no APFS note.

## Acceptance Criteria

- [x] Test (core): `ddc rules --json` lists `trash` in the "Trash" group and in no other. — evidence: `crates/ddc-cli/tests/rules_cli.rs` (`members("Trash") == ["trash"]`), `crates/ddc-core/tests/rules.rs` (`group_of("trash")`), golden `bench/golden/rules.json`
- [x] Test (core): a scan of a temp HOME with a `.Trash` of known size reports the `trash` finding as not selected, and `selected_reclaimable_bytes` leaves its bytes out. — evidence: `crates/ddc-core/tests/report.rs::the_trash_is_found_but_never_selected_or_counted_as_reclaimable`
- [x] Test (DDCKit): `ReviewItems` puts the Trash finding in its own sidebar entry named "Trash" in the review section, and its row cannot be selected; `ReportSummary.reclaimable(including:)` leaves it out even with `.review` included. — evidence: `TrashTests.swift` `sidebar()`, `layout()`, `totals()`
- [x] Test (DDCKit): `TypesRows.make` over the updated `rules.json` fixture gives no "Trash" row, and the other rows are unchanged. — evidence: `SettingsTests.swift` `typesChips` (14 rows, `row("Trash") == nil`)
- [x] Test (DDCKit): a clean result with trashed items says how many bytes went to the Trash; one with deleted items only does not. — evidence: `CleanResultTests.swift` `trashed()`
- Manual checks (1) moved to [Owner checks: manual acceptance across stories](../../../ops/owner-checks-manual-acceptance-across-stories.md) on 2026-10-30, by the owner's decision.

## Final Summary

**What changed** (code commit a1f010f):
- **Core.** The `trash` rule moved from "Your data" to its own "Trash" group in `rules/listing.rs`. Its class, removal and id are unchanged, so the safety model is untouched.
- **Review window.** The Trash's entry under Review says what the Trash is, has an **Open the Trash** button and no Empty button, and has nothing to select. It is found by rule id, not label.
- **Clean result.** It says "N went to the Trash. It comes back when you empty the Trash." with the same button. The APFS note shows only when something was deleted or a tool ran.
- **Duplicate entry.** `trash` left the Swift user-data naming. Otherwise the sidebar would have shown "Trash" twice.

**Tests run:**
- the full gate at commit;
- `swift test` in DDCKit (273 passed);
- the golden re-record (only `rules.json` changed);
- an unsigned Debug app build.

The reviewer said commit. Its nit was applied (the fixture category).

**Beyond the story's text.** The APFS note also shows after a tool command, which frees space now. A clean that only moved items to the Trash or only skipped them shows no note.

**Follow-ups:**
- AC6 is manual for the owner.
- A large Trash sits high in the Review section, because entries are sorted by size there.
- `OpenTrashButton` ignores the result of `NSWorkspace.open`; AC6 shows whether that matters.

## Dependencies

- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model]] §1: the Trash is report only; this story keeps it so
- "Review window: sorted sidebar, Select all and folder groups" (DDC-002): the sidebar entries
- "Review and clean window" (DDC-002): the result view
- "Review window: Clean is not offered when every item would be skipped" (DDC-002): the other stories from the same recording

## Attachments

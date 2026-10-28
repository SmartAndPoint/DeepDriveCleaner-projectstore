---
type: story
id: "story-review-window-after-a-clean-only-skips-that-can-clear-stay-selected"
epic: "DDC-002"
title: "Review window: after a clean, only skips that can clear stay selected"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-29
external_refs: {}
tags: [app, review, selection, clean-result, user-report]
code_refs: ["crates/ddc-core/src/report/after_clean.rs", "crates/ddc-cli/src/cmd/clean_items.rs", "crates/ddc-core/tests/after_clean.rs", "crates/ddc-cli/tests/clean_items_cli.rs", "app/Packages/DDCKit/Sources/DDCKit/AfterClean.swift", "app/Packages/DDCKit/Sources/DDCKit/SkipSummary.swift", "app/Packages/DDCKit/Sources/DDCKit/CleanResult.swift", "app/Packages/DDCKit/Sources/DDCKit/ReviewModel.swift", "app/Packages/DDCKit/Sources/DDCKit/CleanRun.swift", "app/Packages/DDCKit/Tests/DDCKitTests/AfterCleanTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/RealCLICleanTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ContractTests.swift", "app/DeepDriveCleaner/CleanResultView.swift", "app/DeepDriveCleaner/ConfirmSheet.swift", "app/DeepDriveCleaner/ReviewView.swift"]
specs: []
started_at: "2026-10-28T18:28:37.082Z"
closed_at: null
plan_updated_at: "2026-10-28T18:28:37.082Z"
---

# Review window: after a clean, only skips that can clear stay selected

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

From a user's screen recording of 2026-10-28 (app 0.4.0). After a clean of 35 items the result said "0 deleted · 25 to the Trash · 1 tool command · 9 skipped · 1 failed", with the causes "Google Chrome and Visual Studio Code were running (6 items)", "Changed since the scan (2 items)", "In use by node (1 item)", and the Docker prune commands under Failed. It ended with "They stay selected for the next clean." The footer then said "10 items selected · 41.0 GiB". The user cleaned again, and again, and the same items came back each time: the two changed folders, the Docker item, the caches of the running apps.

Today every item that is not gone stays selected (`ReviewModel.reload()`: the selection is kept by `(path, dev, ino)` for every row still in the report). That is right for a skip that can clear by itself or by one step of the user's, and wrong for the others. The table is per outcome of the clean (`Outcome` in `exec/mod.rs`), and per skip reason inside `skipped`:

- **Can clear, stays selected:** `owner-running` (quit the app), `in-use` and `in-use-unknown-cwd` (the process ends), `tool-not-ready` (start Docker Desktop; story "VM images: Docker prune runs only while Docker Desktop answers"). The result says what clears it: "Quit Google Chrome and Visual Studio Code, then clean again."
- **Needs a new scan, deselected:** `changed-since-scan`, `parent-changed`, `missing`, `rule-mismatch`, `not-an-entry`, `entry-locked`, `symlink`, `not-a-directory`, and `incomplete` (the walk was partial; the reason carries no detail that says whether a permission would fix it). The CLI re-checks against the last scan, so the next clean would skip them again. The result says "2 items changed since the scan. Scan Again to see them as they are now." and has a **Scan Again** button that runs the window's own `scanAgain()`.
- **Never cleanable from here, deselected:** `kept`, `protected`, `project-root`, `scan-root`, `outside-scan-roots`, `action-not-allowed`, `needs-confirmation`, `inside-another-item`, `native-unavailable` (the tool is not installed). The result names them as today.
- **A tool command that ran, deselected,** whether it succeeded or failed. A successful prune does not make the folder smaller at once (Docker frees space inside `Docker.raw`, safety ADR §1), so the item is still in the report and would be pruned again. A failed one would fail the same way. The row keeps a short note until the next scan: "Pruned 3 min ago; Docker Desktop shrinks its disk image later" or "The prune failed: <its last line>".
- **A failure of the executor itself** (outcome `failed`), deselected, with its error in the result as today.
- **Not reached** (outcome `not-run`, a stopped clean), stays selected.

The mapping is a single exhaustive switch in DDCKit over `CleanReport.SkipReason` (all 21 cases today, plus `tool-not-ready`), so a new reason cannot be added without choosing. `reload()` runs at other times too (Settings changed, the window became key); the mapping applies only to the reload that follows a clean, with that clean's report in hand. The model keeps the last clean's report for the per-row notes below until the next scan replaces the report. The result's last line changes from "They stay selected for the next clean." to the count that actually stays: "6 items stay selected for the next clean." or nothing when none does.

**Two items "changed since the scan" in every clean.** The CLI edits `last-scan.json` after a clean and leaves the items it did not touch as they were scanned. If what changed them is the clean itself (a folder next to a trashed item, a folder inside the pruned Docker container), deselecting them is a band-aid over a stale report. The first step of this story finds out, on a temp HOME with the real CLI, what makes an untouched item `changed-since-scan` after a clean; if the clean causes it, the fix belongs in the post-clean edit of the report and is added here.

The recording also shows, for one frame, an **empty sheet**: a blank rounded panel between stages. `ConfirmSheet` is already one container whose content switches on `model.sheet?.stage`, so the cause is not a swapped view. More likely it is the `model.sheet == nil` branch drawing an empty padded `Group` during the dismiss animation after `closeSheet()`, or a height change between stages. Reproduce it first; the fix keeps the last stage's content while the sheet is dismissed.

## Decomposition

- [x] Investigate: on a temp HOME, with the real CLI, a clean that trashes one item and leaves a sibling; is the sibling `changed-since-scan` at the next clean? Record the answer in the plan and fix the post-clean report edit if the clean causes it.
- [x] DDCKit: `SkipReason.afterClean` → `.keep` / `.rescan` / `.never`, an exhaustive switch; `ReviewModel.reload()` after a clean keeps only the `.keep` items and the items the clean did not reach (`not-run`, a stopped clean), and drops every item whose tool ran.
- [x] DDCKit: `CleanResult` gains the count that stays selected and a `needsRescan` count; `CleanWording` gets the sentences above.
- [x] DDCKit: a short per-row note for an item whose tool ran in this window's last clean, cleared by the next scan.
- [x] App: `CleanResultView` shows the new last line and the Scan Again button when `needsRescan > 0`; `ConfirmSheet` keeps the last stage's content while `model.sheet` becomes nil during the dismiss.

## Implementation Plan

Planned by `projectstore:planner` on 2026-10-28, after "Review window: Clean is not offered when every item would be skipped" (which adds `AfterClean.swift`).

**The investigation, done while planning.** The built CLI on a temp HOME (no tools on `PATH`, deleted items only): three `__pycache__` folders scanned; one replaced by a fresh folder (new inode); `clean --items` on it and a sibling. The sibling was deleted; the replaced one was `changed-since-scan`, and `last-scan.json` still held its old inode. A second clean skipped it again. Cause: in `ddc-cli/src/cmd/clean_items.rs::run`, the pruned walk's fresh findings (`scanned`) are dropped (`base.unwrap_or(scanned)`), and `after_clean::edit` leaves skipped items as they were. The saved report keeps the stale `(dev, ino)`, and the app reloads it instead of scanning, so every clean skips the item again until a full scan. The clean itself does not change untouched items. The full walk does not loop.

1. **Core fix.** Keep `scanned` in the pruned branch and pass it to `after_clean::edit` as `fresh`. For each listed item that is not user data and was not removed: a finding at its path in `fresh` replaces the base finding (identity, parents, class, sizes), the totals move by the difference as `Edit::resize` does; no finding there and the outcome `missing`, `changed-since-scan`, `rule-mismatch`, `not-a-directory` or `symlink` → the finding is dropped. Picked entries (folder `contents`) are left alone. `tests/after_clean.rs::skipped_failed_and_not_run_items_stay_…` changes with it; a regression test in `ddc-cli/tests/clean_items_cli.rs` (rename a listed folder aside, recreate it, clean: the saved report holds the new inode and a second clean deletes it).
2. **The selection.** `CleanRun.runClean` keeps the last clean's report and end time on `ReviewModel`, with a flag the next `reload()` consumes: a row in the report stays selected only for `skipped` with `afterClean == .keep`, or `not-run`; native items (any result), `failed`, and rescan or never skips are deselected; rows outside the report keep their selection. Other reloads (Settings, Scan Again) do not consume it.
3. **The tool note.** `ReviewModel.toolNote(for:)` from the last clean, shown in the row detail (`ReviewView`, `ReviewGroupsView`); cleared when a scan newer than the clean's end replaces the report, as `cleaned` is.
4. **The result.** `CleanResult` gains `staysSelected` and `needsRescan`; `CleanWording` the sentences. `CleanResultView`: the count line or none; **Scan Again** when `needsRescan > 0`, which closes the sheet and runs `scanAgain()`, disabled while cleaning or rescanning.
5. **The empty sheet.** `ReviewModel.lastSheet`, kept after `closeSheet()`; `ConfirmSheet` draws `model.sheet ?? model.lastSheet`, so the dismiss animation never shows an empty panel. The presentation binding stays on `model.sheet != nil`.
6. **Tests.** New `AfterCleanTests` (every tag maps; a version-3 fake report with the eight kinds leaves owner-running, in-use and not-run selected; the result lines; the note cleared by a newer scan; `lastSheet` after close). `ContractTests`: Scan Again calls only `scanAgain()`. New `RealCLICleanTests`, a copy of the `RealCLIDryRunTests` wrapper with an empty `PATH`, listing deleted items only (never trashed ones, which would use the real Trash), replaying the loop.

## Acceptance Criteria

- [x] Test (DDCKit): `afterClean` covers every `SkipReason` case (an exhaustive switch; a test lists the CLI's reason tags from `CleanReport` and maps each). — evidence: `SkipSummaryTests.afterCleanTable` (every tag of `SkipReason.tags`, 22), the switch in `AfterClean.swift` has no `default`
- [x] Test (DDCKit): a fake clean report with one item of each kind — owner-running, in-use, changed-since-scan, kept, a native item that succeeded, a native item that failed, a `failed` item, a `not-run` item — leaves selected exactly the owner-running, in-use and not-run items. — evidence: `AfterCleanTests.selection`, `AfterCleanTests.otherReloads`
- [x] Test (DDCKit): the result says "3 items stay selected for the next clean.", and offers Scan Again because one item changed since the scan; with nothing kept, the line is absent. — evidence: `AfterCleanTests.resultLines`
- [x] Test (DDCKit): after a successful prune of the Docker item, the item is deselected and its note says it was pruned; a new scan clears the note. — evidence: `AfterCleanTests.toolNotes`
- [x] Contract test (source check): `CleanResultView`'s Scan Again calls only the model's `scanAgain()`. — evidence: `ContractTests.resultScanAgain`
- [x] Test (DDCKit, real CLI on a temp HOME, in the pattern of the existing real-CLI dry-run tests): select a cache and a folder, change the folder after the scan, clean, reload: the cache is gone, the folder is deselected, and the result offers Scan Again. This replays the recording's loop end to end. — evidence: `RealCLICleanTests.changedFolderLoop` (empty `PATH`, deletes only); core: `crates/ddc-core/tests/after_clean.rs::skipped_and_not_run_items_take_the_rechecks_finding_or_leave`, `crates/ddc-cli/tests/clean_items_cli.rs::a_skipped_item_is_saved_as_the_recheck_found_it`
- [x] Test (DDCKit): with `model.sheet` set to nil while the sheet is presented, the sheet's view model still yields the last stage's content (no empty state). — evidence: `AfterCleanTests.lastSheet`
- [ ] Manual (owner): reproduce the recording's mix (a running app's cache, a folder changed after the scan, the Docker item with Docker Desktop quit), clean, and check what stays selected and what the result says; the sheet never shows an empty panel between its stages.

## Final Summary

**What changed (code commit bb26f4c).**
- **Core.** The cause of the recording's loop was the report saved after a clean, as the investigation found. `after_clean::edit` now takes the re-check's walk (`fresh`) when it edits an older report. A listed item the clean skipped or did not reach takes the re-check's finding at its path: new identity and size, with totals moved by the difference. When the re-check has no finding there and the skip says the item is gone from the disk as scanned (`missing`, `changed-since-scan`, `rule-mismatch`, `not-a-directory`, `symlink`), the finding leaves. A folder renamed away is reported as `changed-since-scan`. `failed` items and picked entries are left as they were. The lines under a refreshed item leave until the next full scan. Decision 6 of "Faster clean: re-check only what the selection needs" (DDC-002) got a dated note.
- **Selection.** `runClean` keeps the clean's report on `ReviewModel` (`lastClean`), and only the reload after the clean applies it. A row in the report stays selected for a `keep` skip (`AfterClean.swift`, the table shared with "Review window: Clean is not offered when every item would be skipped") or `not-run`. Everything else in the report is deselected. Rows outside it keep their tick.
- **Result.** `CleanResult.staysSelected` and `needsRescan`. The last line is "N items stay selected for the next clean." or nothing. The advice ("Quit A, then clean again.", "N items changed since the scan. Scan Again to see them as they are now.") shares its helpers with the sheet. **Scan Again** closes the sheet and runs `scanAgain()`. It is disabled while cleaning, rescanning or with scans paused.
- **Tool note.** A tool-cleaned row shows "Pruned 3 min ago; Docker Desktop shrinks its disk image later" or "The prune failed: <its last line>" until a newer scan. The last non-empty line is used because docker's first line is often a header.
- **Empty sheet.** `ReviewModel.lastSheet` is kept after `closeSheet()`, and `ConfirmSheet` draws `model.sheet ?? model.lastSheet`. The flash was not reproduced first. The fix targets the dismiss animation, and the manual check decides.

**Tests executed.** nextest 430/430 in `make gate-fast` and the pre-push `make gate`. `swift test`: 269 tests in 32 suites, the real-CLI loop included. The app target builds unsigned. The reviewer (`projectstore:reviewer`) found no blockers. Its should-fix (the module doc and decision 6) and three nits were applied: the lines under a refreshed item, "see it as it is now" for one item, and `selectAfterClean` reset when a clean starts.

**Follow-ups.**
- A plain reload between the after-clean rescan and `runClean`'s resume could clear `lastClean` first. The selection would then stay as before. This is a narrow race.
- In a scoped clean, "N stay selected" counts only the clean's items.
- An item swapped between the re-check and the action keeps its old identity for one more clean.

**Open.** The manual criterion is the owner's.

## Dependencies

- "Review and clean window" (DDC-002): the selection and the result view
- "Review window: clean only the selection in this group" (DDC-002): the scoped clean, which shares `reload()`
- "VM images: Docker prune runs only while Docker Desktop answers" (DDC-002): `tool-not-ready`, a reason that stays selected
- "Review window: Clean is not offered when every item would be skipped" (DDC-002)

## Attachments

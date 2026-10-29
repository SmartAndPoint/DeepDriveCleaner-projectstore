---
type: story
id: "story-whole-disk-free-the-checked-areas-in-one-sheet"
epic: "DDC-002"
title: "Whole disk: free the checked areas in one sheet"
status: in-progress
priority: p3
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-29
external_refs: {}
tags: [app, whole-disk, free, user-report]
code_refs: ["app/DeepDriveCleaner/AreaActionSheet.swift", "app/DeepDriveCleaner/DiskView.swift", "app/Packages/DDCKit/Sources/DDCKit/AreaActions.swift", "app/Packages/DDCKit/Sources/DDCKit/DiskFreeRun.swift", "app/Packages/DDCKit/Sources/DDCKit/DiskModel.swift", "app/Packages/DDCKit/Sources/DDCKit/DiskSummary.swift", "app/Packages/DDCKit/Tests/DDCKitTests/CheckedFreeTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ContractTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/DiskFreeTests.swift", "crates/ddc-cli/tests/disk_free_cli.rs"]
specs: []
started_at: "2026-10-29T13:12:54.962Z"
closed_at: null
plan_updated_at: "2026-10-29T13:12:54.962Z"
---

# Whole disk: free the checked areas in one sheet

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
| **Priority** | p3 |
| **Assignee** | Evgenii Konev |

---

## Description

Requested by the owner on 2026-10-28, after a user's screen recording (app 0.4.0): the Whole disk view, like User data, has no "delete selected" button.

In the Whole disk view only rows with an offer can free anything, and only through their owning tool (story "Whole disk: free space with the owning tool" and its successors). An offer exists, from `AreaActions.offers`, for Homebrew (**Clean Up Homebrew…**, `brew-cleanup`), each simulator runtime that can be deleted now (**Delete Runtime…**, `simctl runtime delete`), each writable `/opt` conda environment (**Remove Environment…**, `conda-env-remove`) and each writable conda package cache (**Clean Package Cache…**, `conda-clean`, on the Conda row or each install's row). Every other row is measured only; that is the safety ADR's rule for areas outside the home folder, and this story does not change it. A machine with several old runtimes or environments needs one sheet per row today.

- **Checkboxes.** A row gets a checkbox exactly when it has an offer (`DiskSummary.Row.offer != nil`), unticked by default. No row can be checked while the view cannot free (`!canFree`: during a scan, a run, or while a sheet is open). The checks are pruned to the offers of each new disk report, so a row that left the list is not counted.
- **Package removals never join.** `brew-uninstall` and `brew-uninstall-orphans` live only in the packages sheet, which has no checkboxes; `toggle` refuses those two ids too (menu bar ADR §6 item 11).
- **The bar** above the list says "2 checked" with a **Free 2 Checked…** button. It shows no size: a row's size is not what its action frees (Homebrew's row is all of `/opt/homebrew`, `brew cleanup` frees only old versions and downloads; the Conda row is the whole area, or an install's row its cache with what its environments share, while `conda clean` removes only packages no environment uses), and ddc never reports space it did not free (safety ADR, 2026-10-28 amendment). The honest estimate is the sheet's, from the tools' own dry runs. The button is disabled while `!canFree` or with nothing checked. A check survives a Measure while its row is still offered (checks are keyed by `Offer.id`).
- **The sheet for a list.** The button opens the area-action sheet for the list: one `--dry-run` call with the whole list, then each action's preview in run order, with its consequence (what it cannot undo), its estimate, and its exact command; then one confirmation. While the dry run runs, the sheet names how many actions it checks (each conda action asks conda on its own, so a list can take tens of seconds). The total sums only the would-run previews' estimates, worded "about"; when a would-run preview has no estimate, the total says "at least". While it runs, the sheet says "Freeing 2 areas…" (the CLI reports no per-action progress) and that Stop ends the one running and leaves the rest not run. After the run the sheet shows each action's result (`FreeWording.result`) and no total: ddc reports space per action, as the CLI measured it. A list of one keeps today's layout, except that a `failed` preview shows its reason and no confirm button, as in the list.
- **Run order.** The list runs in the view's row order, except that every `conda-clean` moves after every `conda-env-remove`, so a removal and its install's cache clean run removal first. A clean previewed before a removal in the same list understates its estimate; the sheet never overstates.
- **Mixed dry runs.** When some previews come back `would-run` and others refused or failed (a runtime booted meanwhile), the sheet lists each with its reason, and the confirm button runs only the `would-run` ones, saying so: "Free 1 of 2". The list the CLI gets on confirm is exactly those, rebuilt from the sheet's previews, never from the current checks; the CLI re-checks each anyway. With none `would-run`, there is nothing to confirm. A row refused in the preview gets no row result, as today.
- A Stop during the run stops the action that runs; the ones not started are reported "not run", as the CLI does today.
- After the run, each row says its own result, as today, and the sheet's rows are unchecked, whatever the result.

The one-row buttons stay. It is p3: the gain is one sheet instead of a few; the owner's request for User data is covered at p2 by "User data: move a folder's picked files to the Trash from that folder". The sheet stays the one caller of `freeConfirmedActions()` (menu bar ADR §6 item 11), now with a list. No schema changes: the free list is at `schema_version` 3 since the Homebrew story (2 since the conda story), and the CLI already validates, previews and runs a list of any length in order under one lock, journaling one line per action.

## Decomposition

- [x] DDCKit: `FreeSheet` holds `offers`, and its `confirm` and `result` stages hold one report per offer, aligned by index; `previewAction`'s "closed meanwhile" check compares the list; `runFree` writes them all and accepts a report only when it has one action per offer with the same `action` and `identifier` in the same order (today it requires exactly one); `brewPackages.actionEnded` is called per pair.
- [x] DDCKit: `DiskModel.checked`, `toggle`, `canCheck`, the run order, `previewChecked()`, the would-run subset on confirm, the "Free 1 of 2" wording.
- [x] App: checkboxes on offer rows, the bar and its button in `DiskView`; `AreaActionSheet` shows a list of previews and results.

## Implementation Plan

From the planner's pass of 2026-10-29 at main 1119a97. App only; no Rust change, no schema change.

1. **Lists without a behaviour change** (`DiskModel.swift`, `DiskFreeRun.swift`): `FreeSheet.offers` and the array stages; `runFree` over `[Offer]` with the count, action and identifier check by index (a mismatch is `.failed`, never a partial result); `previewAction(_:)` delegates to a private `preview(_ offers:)`; `actionEnded` per pair; `AreaActionSheet` uses the first offer for now. The existing `DiskFreeTests` and `BrewPackagesTests` follow the array stages; add the mismatch test.
2. **Checked state and order** (`DiskModel.swift`, `AreaActions.swift`, `DiskFreeRun.swift`): `checked: Set<String>` (`public internal(set)`), pruned in a `didSet` on `report` (so `apply`, `run` and `load`'s `.none` all prune); `toggle` guarded by `canFree`, the offer being among the current offers, and `joinsList` (false for the two package ids); `canCheck`; `runOrder` (row order from one factored `AreaActions.orderedOffers(report)` that `DiskSummary.make` also uses, so rows and run order cannot disagree; `conda-clean` stably moved last); `checkedOffers`. Tests for refused toggles and the order.
3. **The checked run** (`DiskFreeRun.swift`): `previewChecked()` as the bar's entry; `freeConfirmedActions()` keeps its name and no arguments, runs only the offers whose preview is `would-run` (at least one), and clears the sheet's offers from `checked` after the run; the confirm title "Free 2" / "Free 1 of 2" (a list of one keeps `offer.confirmTitle`). Tests: AC1, AC2, AC4; the never-frees regex in `ContractTests` (ScheduledWork, PopoverModel, AppDelegate, SchedulerAgent, ScanDue) gains `previewChecked`; `freeFailure`'s refused text says "the list" for a list.
4. **App** (`DiskView.swift`, `AreaActionSheet.swift`, `ContractTests.swift`): `freeCallers` asserts `.previewChecked(` appears only in `DiskView.swift`; the `CheckBox` (from `ReviewView.swift`) on `DiskRow` when the row has an offer, enabled by `canCheck`; the bar above the rows (like the Review window's `SelectionFooter`); the list layout in the sheet (per offer its name, consequence, estimate, preview and command, or its reason; the total of the estimates; Cancel and the confirm button, absent when nothing would run; per-action results with `FreeWording.result`, the Terminal command and output). One `.sheet` stays; no new runner kind.
5. **CLI test** (`crates/ddc-cli/tests/disk_free_cli.rs`): two runtimes in one list (a second asset folder, a `deleted` marker per runtime) give two journal lines, and both leave `last-disk.json` with `listed_bytes` recomputed.
6. Vault: `projectstore:reviewer`, evidence, code_refs.

Pitfalls: never iterate the `Set` for order; never send refused or failed previews: the CLI re-probes, and one whose refusal cleared meanwhile would run unconfirmed; keep the brew packages flow a list of one; the lock is held up to one `NATIVE_TIMEOUT` per action, so the running text promises no speed.

## Acceptance Criteria

- [x] Test (DDCKit, fake runner): two runtimes checked → one dry-run call with both in the list, then one run call with both; the results map back to their rows; the result stage shows no summed figure; a report whose actions do not match the list in count, order or identifier is a failure, not a partial success. — evidence: `app/Packages/DDCKit/Tests/DDCKitTests/CheckedFreeTests.swift` `freeBoth`, `mismatchedRun`; `DiskFreeTests.mismatchedReport`
- [x] Test (DDCKit): a stop after the first action leaves the second "not run"; after a run, whatever its result, a failed report included, the sheet's checkboxes are clear. — evidence: `app/Packages/DDCKit/Tests/DDCKitTests/CheckedFreeTests.swift` `stopAndFailure`, `mismatchedRun` (checks clear after a failed run)
- [x] Test (DDCKit): rows without an offer, a package removal, and every row while `!canFree` (during a scan, while a sheet is open) cannot be checked; checks of rows no longer offered are pruned when the report changes, including to none or unreadable; the bar's button is disabled while `!canFree` or with nothing checked. — evidence: `app/Packages/DDCKit/Tests/DDCKitTests/CheckedFreeTests.swift` `refusedChecks`, `pruned`, `rowOrder`; `DiskView.swift` `.disabled(!model.canFreeChecked)`
- [x] Test (DDCKit): of two previews, one `would-run` and one refused → the confirm runs a list of one, its title says "Free 1 of 2", and the refused one shows its reason and gets no entry in `freeResults`; with none `would-run`, confirm is not offered and no run call is made. — evidence: `app/Packages/DDCKit/Tests/DDCKitTests/CheckedFreeTests.swift` `mixedPreviews`, `totalAndRefusal`, `listOfOne`
- [x] Test (DDCKit): a conda environment and its install's package cache checked together produce a list with `conda-env-remove` first. — evidence: `app/Packages/DDCKit/Tests/DDCKitTests/CheckedFreeTests.swift` `condaOrder`
- [x] Contract test: `freeConfirmedActions()` keeps one caller, the sheet's confirm button, and the checked bar's button is the only caller of `previewChecked()`. — evidence: `ContractTests.freeCallers` (one caller of `.freeConfirmedActions(`, `.previewChecked(` once in `DiskView.swift`, one `.sheet(`, never-frees regex with `previewChecked`)
- [x] Test (CLI): two runtimes in one `--actions` list give two journal lines and both leave `last-disk.json`. — evidence: `crates/ddc-cli/tests/disk_free_cli.rs` `two_runtimes_in_one_list_run_in_order_each_journaled_and_both_leave_the_saved_disk_report`
- [ ] Manual (owner): with two deletable runtimes, check both, free them in one sheet, and compare with `xcrun simctl runtime list`.

## Final Summary

App only, five commits on main, from "hold a list of area actions in the free sheet" to "free two runtimes in one disk free list, as the checked rows send them". No schema change: free stays 3, disk stays 2.

- **What changed.**
  - Every Whole disk row with an offer has a checkbox. The bar reads "N checked" and holds **Free N Checked…**.
  - The area-action sheet takes a list. It makes one dry-run call and shows each preview in run order, with its consequence, estimate and command. The total reads "about" or "at least".
  - The confirm sends only the would-run subset ("Free 1 of 2"), rebuilt from the sheet's own previews. Each action gets its own result, and no total is shown after the run.
  - Rows and run order come from one walk (`AreaActions.places`), with `conda-clean` stably last.
  - Checks are pruned in a `didSet` on the report.
  - A failed dry run of a single non-brew action now shows its output and no confirm button. Before, the sheet showed a confirm button that did nothing.
- **Tests.** `make gate` passed. `swift test` passed 320 tests. The CLI test runs two runtimes in one list. The app builds unsigned.
- **Reviews.** The critic approved the plan with fixes, and they were applied. `projectstore:reviewer` found no consent-safety defect: only previews shown as would-run are ever sent, in the order shown. Its nits are left as they are: the sheet's own wording is untested, `orderedOffers` is rebuilt per row, and the caller count of `freeConfirmedActions`.
- **Deviations at implementation.**
  - `conda-clean` runs after every other action, not only after `conda-env-remove`.
  - After confirm, the sheet's offers shrink to the would-run subset, so the results list only what ran.
  - Checks are also disabled while the Homebrew packages sheet is open.
- **Open.** The owner's manual check. VoiceOver on the row checkbox inside the combined row element is worth a look.

## Dependencies

- "Whole disk: free space with the owning tool" (DDC-002): the actions, the sheet and the `disk free` list
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]] §6 item 11
- "User data: move a folder's picked files to the Trash from that folder" (DDC-002): the same request for User data

## Attachments

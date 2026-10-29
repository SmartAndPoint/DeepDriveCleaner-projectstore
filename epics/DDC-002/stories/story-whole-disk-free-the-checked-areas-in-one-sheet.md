---
type: story
id: "story-whole-disk-free-the-checked-areas-in-one-sheet"
epic: "DDC-002"
title: "Whole disk: free the checked areas in one sheet"
status: planned
priority: p3
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [app, whole-disk, free, user-report]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Whole disk: free the checked areas in one sheet

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p3 |
| **Assignee** | Evgenii Konev |

---

## Description

Requested by the owner on 2026-10-28, after a user's screen recording (app 0.4.0): the Whole disk view, like User data, has no "delete selected" button.

In the Whole disk view only two kinds of row can free anything, and only through their owning tool (story "Whole disk: free space with the owning tool"): each deletable simulator runtime (**Delete Runtime…**, `simctl runtime delete`) and Homebrew (**Clean Up Homebrew…**, `brew cleanup`). Every other row (Applications, the shared Library, macOS's downloaded data, `/usr/local`, temporary folders, the sleep image, the other users) is measured only; that is the safety ADR's rule for areas outside the home folder, and this story does not change it. A machine with several old runtimes needs one sheet per runtime today.

- Each row with an offer gets a checkbox, unticked by default. Rows without an offer get none (an offer exists only for a runtime that can be deleted now, `AreaActions.offersDelete`), and no row can be checked while the view cannot free (`!canFree`, during a scan or a run).
- A bar above the list says "2 checked · 14.2 GiB" with a **Free 2 Checked…** button.
- The button opens the area-action sheet for the list: every action's own dry run and exact command, one after another, then one confirmation. The CLI already takes a list (`disk free --actions` with several entries) and runs them in order under one lock; each action's result is reported apart.
- **Mixed dry runs.** When some previews come back `would-run` and others refused or failed (a runtime booted meanwhile), the sheet lists each with its reason, and the confirm button runs only the `would-run` ones, saying so: "Free 1 of 2". The list the CLI gets on confirm is exactly those; it re-checks each anyway. With none `would-run`, there is nothing to confirm.
- A Stop during the run stops the action that runs; the ones not started are reported "not run", as the CLI does today.
- After the run, each row says its own result, as today, and its checkbox is cleared.

**Since this story was written** (2026-10-29), story "Whole disk: Conda environments and its package cache, with conda" landed two more offers: **Remove Environment…** on each `/opt` environment row (`conda-env-remove`) and **Clean Package Cache…** on the Conda row or each install's row (`conda-clean`). Both get a checkbox like the others; a list that removes an environment and cleans its install's cache runs them in the list's order, the removal first. Story "Whole disk: Homebrew packages, their sizes, and brew uninstall" adds `brew-uninstall` and `brew-uninstall-orphans`, which never join a multi-row free (menu bar ADR §6 item 11): their rows get no checkbox. The free list is at `schema_version` 2 since the conda story.

The one-row buttons stay. It is p3 because only runtimes and Homebrew can free anything here, so the gain is one sheet instead of a few; the owner's request for User data is covered at p2 by "User data: move a folder's picked files to the Trash from that folder". The sheet stays the one caller of `freeConfirmedActions()` (menu bar ADR §6 item 11), now with a list.

## Decomposition

- [ ] DDCKit: `DiskModel` gains `checked: Set<AreaActions.Offer.ID>`; `FreeSheet` holds a list of offers; `runFree` writes them all and accepts a report with one action per offer in the same order (today it requires exactly one).
- [ ] App: checkboxes on offer rows, the bar and its button in `DiskView`; `AreaActionSheet` shows a list of previews and results.

## Implementation Plan

## Acceptance Criteria

- [ ] Test (DDCKit, fake runner): two runtimes checked → one dry-run call with both in the list, then one run call with both; the results map back to their rows; a report whose actions do not match the list in count or order is a failure, not a partial success.
- [ ] Test (DDCKit): a stop after the first action leaves the second "not run"; checkboxes clear after a run.
- [ ] Test (DDCKit): rows without an offer, and every row while `!canFree`, cannot be checked.
- [ ] Test (DDCKit): of two previews, one `would-run` and one refused → the confirm runs a list of one, and the refused one shows its reason; with none `would-run`, confirm is not offered.
- [ ] Contract test: `freeConfirmedActions()` keeps one caller, the sheet's confirm button.
- [ ] Manual (owner): with two deletable runtimes, check both, free them in one sheet, and compare with `xcrun simctl runtime list`.

## Final Summary

## Dependencies

- "Whole disk: free space with the owning tool" (DDC-002): the actions, the sheet and the `disk free` list
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]] §6 item 11
- "User data: move a folder's picked files to the Trash from that folder" (DDC-002): the same request for User data

## Attachments

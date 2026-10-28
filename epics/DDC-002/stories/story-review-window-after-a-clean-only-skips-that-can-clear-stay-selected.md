---
type: story
id: "story-review-window-after-a-clean-only-skips-that-can-clear-stay-selected"
epic: "DDC-002"
title: "Review window: after a clean, only skips that can clear stay selected"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [app, review, selection, clean-result, user-report]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Review window: after a clean, only skips that can clear stay selected

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

From a user's screen recording of 2026-10-28 (app 0.4.0). After a clean of 35 items the result said "0 deleted · 25 to the Trash · 1 tool command · 9 skipped · 1 failed", with the causes "Google Chrome and Visual Studio Code were running (6 items)", "Changed since the scan (2 items)", "In use by node (1 item)", and the Docker prune commands under Failed. It ended with "They stay selected for the next clean." The footer then said "10 items selected · 41.0 GiB". The user cleaned again, and again, and the same items came back each time: the two changed folders, the Docker item, the caches of the running apps.

Today every item that is not gone stays selected (`ReviewModel.reload()`: the selection is kept by `(path, dev, ino)` for every row still in the report). That is right for a skip that can clear by itself or by one step of the user's, and wrong for the others. The table is per outcome of the clean (`Outcome` in `exec/mod.rs`), and per skip reason inside `skipped`:

- **Can clear, stays selected:** `owner-running` (quit the app), `in-use` and `in-use-unknown-cwd` (the process ends), `tool-not-ready` (start Docker Desktop; story "VM images: Docker prune runs only while Docker Desktop answers"). The result says what clears it: "Quit Google Chrome and Visual Studio Code, then clean again."
- **Needs a new scan, deselected:** `changed-since-scan`, `parent-changed`, `missing`, `rule-mismatch`, `not-an-entry`, `entry-locked`, `symlink`, `not-a-directory`, and `incomplete` (the walk was partial; the reason carries no detail that says whether a permission would fix it). The CLI re-checks against the last scan, so the next clean would skip them again. The result says "2 items changed since the scan. Scan Again to see them as they are now." and has a **Scan Again** button that runs the window's own `scanAgain()`.
- **Never cleanable from here, deselected:** `kept`, `protected`, `project-root`, `scan-root`, `outside-scan-roots`, `action-not-allowed`, `needs-confirmation`, `inside-another-item`, `native-unavailable` (the tool is not installed). The result names them as today.
- **A tool command that ran, deselected,** whether it succeeded or failed. A successful prune does not make the folder smaller at once (Docker frees space inside `Docker.raw`, safety ADR §1), so the item is still in the report and would be pruned again. A failed one would fail the same way. The row keeps a short note until the next scan: "Pruned 3 min ago; Docker Desktop shrinks its disk image later" or "The prune failed: <first line>".
- **A failure of the executor itself** (outcome `failed`), deselected, with its error in the result as today.
- **Not reached** (outcome `not-run`, a stopped clean), stays selected.

The mapping is a single exhaustive switch in DDCKit over `CleanReport.SkipReason` (all 21 cases today, plus `tool-not-ready`), so a new reason cannot be added without choosing. `reload()` runs at other times too (Settings changed, the window became key); the mapping applies only to the reload that follows a clean, with that clean's report in hand. The model keeps the last clean's report for the per-row notes below until the next scan replaces the report. The result's last line changes from "They stay selected for the next clean." to the count that actually stays: "6 items stay selected for the next clean." or nothing when none does.

**Two items "changed since the scan" in every clean.** The CLI edits `last-scan.json` after a clean and leaves the items it did not touch as they were scanned. If what changed them is the clean itself (a folder next to a trashed item, a folder inside the pruned Docker container), deselecting them is a band-aid over a stale report. The first step of this story finds out, on a temp HOME with the real CLI, what makes an untouched item `changed-since-scan` after a clean; if the clean causes it, the fix belongs in the post-clean edit of the report and is added here.

The recording also shows, for one frame, an **empty sheet**: a blank rounded panel between stages. `ConfirmSheet` is already one container whose content switches on `model.sheet?.stage`, so the cause is not a swapped view. More likely it is the `model.sheet == nil` branch drawing an empty padded `Group` during the dismiss animation after `closeSheet()`, or a height change between stages. Reproduce it first; the fix keeps the last stage's content while the sheet is dismissed.

## Decomposition

- [ ] Investigate: on a temp HOME, with the real CLI, a clean that trashes one item and leaves a sibling; is the sibling `changed-since-scan` at the next clean? Record the answer in the plan and fix the post-clean report edit if the clean causes it.
- [ ] DDCKit: `SkipReason.afterClean` → `.keep` / `.rescan` / `.never`, an exhaustive switch; `ReviewModel.reload()` after a clean keeps only the `.keep` items and the items the clean did not reach (`not-run`, a stopped clean), and drops every item whose tool ran.
- [ ] DDCKit: `CleanResult` gains the count that stays selected and a `needsRescan` count; `CleanWording` gets the sentences above.
- [ ] DDCKit: a short per-row note for an item whose tool ran in this window's last clean, cleared by the next scan.
- [ ] App: `CleanResultView` shows the new last line and the Scan Again button when `needsRescan > 0`; `ConfirmSheet` keeps the last stage's content while `model.sheet` becomes nil during the dismiss.

## Implementation Plan

## Acceptance Criteria

- [ ] Test (DDCKit): `afterClean` covers every `SkipReason` case (an exhaustive switch; a test lists the CLI's reason tags from `CleanReport` and maps each).
- [ ] Test (DDCKit): a fake clean report with one item of each kind — owner-running, in-use, changed-since-scan, kept, a native item that succeeded, a native item that failed, a `failed` item, a `not-run` item — leaves selected exactly the owner-running, in-use and not-run items.
- [ ] Test (DDCKit): the result says "3 items stay selected for the next clean.", and offers Scan Again because one item changed since the scan; with nothing kept, the line is absent.
- [ ] Test (DDCKit): after a successful prune of the Docker item, the item is deselected and its note says it was pruned; a new scan clears the note.
- [ ] Contract test (source check): `CleanResultView`'s Scan Again calls only the model's `scanAgain()`.
- [ ] Test (DDCKit, real CLI on a temp HOME, in the pattern of the existing real-CLI dry-run tests): select a cache and a folder, change the folder after the scan, clean, reload: the cache is gone, the folder is deselected, and the result offers Scan Again. This replays the recording's loop end to end.
- [ ] Test (DDCKit): with `model.sheet` set to nil while the sheet is presented, the sheet's view model still yields the last stage's content (no empty state).
- [ ] Manual (owner): reproduce the recording's mix (a running app's cache, a folder changed after the scan, the Docker item with Docker Desktop quit), clean, and check what stays selected and what the result says; the sheet never shows an empty panel between its stages.

## Final Summary

## Dependencies

- "Review and clean window" (DDC-002): the selection and the result view
- "Review window: clean only the selection in this group" (DDC-002): the scoped clean, which shares `reload()`
- "VM images: Docker prune runs only while Docker Desktop answers" (DDC-002): `tool-not-ready`, a reason that stays selected
- "Review window: Clean is not offered when every item would be skipped" (DDC-002)

## Attachments

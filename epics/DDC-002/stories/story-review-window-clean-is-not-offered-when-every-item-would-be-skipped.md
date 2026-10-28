---
type: story
id: "story-review-window-clean-is-not-offered-when-every-item-would-be-skipped"
epic: "DDC-002"
title: "Review window: Clean is not offered when every item would be skipped"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [app, review, confirmation, user-report]
code_refs: []
specs: []
started_at: "2026-10-28T18:28:36.859Z"
closed_at: null
plan_updated_at: "2026-10-28T18:28:36.859Z"
---

# Review window: Clean is not offered when every item would be skipped

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

From a user's screen recording of 2026-10-28 (app 0.4.0), reported by the owner as "not everything that is selected gets deleted". It is the first of the backlog made from that recording; the others are listed under Dependencies.

What the recording shows. Google Chrome and Visual Studio Code were running. The user opened Clean 1 from Browser and app caches…, then Clean 5 from IDE caches…. Each confirmation sheet said, in its warnings line, "Nothing can be cleaned right now: every item would be skipped", and listed every row as "Google Chrome is running · will be skipped" (or Visual Studio Code). The sheet's header still said "1.5 GiB selected · frees about 1.5 GiB", and the Clean 1 item button was enabled. The user pressed it, several times over the session. Each result said "Free space 240.2 → 240.2 GiB · 0 deleted · 6 skipped", and the items "stay selected for the next clean". To the user, the app accepted a clean and deleted nothing.

Once the button was pressed while the line said "Waiting for the running scan to check for warnings…": the clean started before the check could say anything.

The sheet already knows each item's prediction (`ConfirmSheet.warningsLine`, `sheet.predictions`, the CLI's `clean --items --dry-run`). This story makes the sheet act on it:

- **Every item would be skipped.** The Clean button is disabled. The warnings line says what to do, by the class each skip has in the table of "Review window: after a clean, only skips that can clear stay selected", so the sheet and the result never give different advice for the same item:
  - a skip that can clear (`owner-running`, `in-use`, `tool-not-ready`): "Quit Google Chrome and Visual Studio Code, then Check Again.", naming the apps from the predictions;
  - a skip that needs a new scan (`changed-since-scan` and the others of that class): "These items changed since the scan. Scan Again to see them as they are now.", since Check Again would skip them again;
  - a skip that never clears here: its cause, in the result view's words.

  With several classes, the line takes them in that order. DeepDriveCleaner never quits an app itself.
- **Some items would be skipped.** The header's "frees about" counts only the items not predicted to be skipped: "6 selected · 1.6 GiB · frees about 0.4 GiB; 1.2 GiB will be skipped". The button keeps its count of the selection, because the selection is what the CLI is given and re-checks.
- **The check is running** (`.checking`, seconds for a usual selection): the Clean button is disabled until it ends. The recording shows the button pressed before the check could speak. A check that runs long (the dry run walks a 38.9 GiB Docker folder) must not trap the user, so the warnings line gets a **Stop** while checking; it ends in `.stopped`, where Clean is enabled. Back to Review stays the other way out, and reopening the sheet checks again.
- **The check waits for a scan** (`.waiting`, the lock is the running scan's, which can take minutes): the button stays enabled, as today, because the CLI re-checks every item before it touches it (safety ADR §7) and a long scan must not hold a clean hostage. The header says "frees about" with "before the check" appended, so the number is not read as a promise. When the check later finishes with every item skipped, the button becomes disabled at once.
- **A check that failed, was stopped, or found the lock held from Terminal** (`.failed`, `.stopped`, `.busy`) keeps the button enabled, as today: the CLI's own re-check decides.

Nothing changes in the CLI. The predictions are the CLI's; the app only stops offering a clean the CLI has already said will do nothing.

## Decomposition

- [ ] Order: this story and "Review window: after a clean, only skips that can clear stay selected" share the `afterClean` table; whichever is implemented first adds it.
- [ ] DDCKit: a pure value on the sheet state, beside `predictions`: `skipSummary` with the count and bytes predicted to be skipped, the owner apps named, and a `nothingToClean` flag (every confirmed item `wouldSkip`, check finished). Unit-tested without a view.
- [ ] DDCKit: `CleanWording` sentences for the warnings line ("Quit A and B, then Check Again.", the most common other cause) and for the header ("frees about X; Y will be skipped", "frees about X before the check").
- [ ] App: `ConfirmSheet` disables the Clean button on `nothingToClean`, and shows the new header and warnings line. `cleanConfirmedItems()` itself returns without running when `nothingToClean` holds or the check is `.checking`; the existing contract test that pins its one caller (`ContractTests`) is extended to the guard.

## Implementation Plan

Planned by `projectstore:planner` on 2026-10-28, after "VM images: Docker prune runs only while Docker Desktop answers". DDCKit first, then the sheet.

1. **The shared table.** New `DDCKit/AfterClean.swift`: `enum AfterClean { keep, rescan, never }` and `SkipReason.afterClean`, one exhaustive switch over all 22 reasons, as the table of "Review window: after a clean, only skips that can clear stay selected" (keep: owner-running, in-use, in-use-unknown-cwd, tool-not-ready; never: kept, protected, project-root, scan-root, outside-scan-roots, action-not-allowed, needs-confirmation, inside-another-item, native-unavailable; rescan: the rest, incomplete included). This story adds it; the other uses it.
2. **The sheet's summary.** `ReviewModel.Sheet` gains `skipSummary` (count, bytes of the `wouldSkip` items, owner apps once each in confirmation order, the classes present, `nothingToClean` = check `.checked` and every item `wouldSkip`) and `cleanAllowed` = confirm stage, not `.checking`, not `nothingToClean`. `Confirmation.reclaimable(excluding:)` for the header.
3. **The guard.** `CleanRun.cleanConfirmedItems()` returns unless `cleanAllowed`. A public `stopCheck()` wraps `stopWarnings()` (it already ends `.checking`/`.waiting` as `.stopped`).
4. **Wording.** `CleanWording`: the warnings sentence by class, keep → rescan → never: "Quit A and B, then Check Again." for owner apps, for `tool-not-ready`, "Start Docker Desktop, then Check Again." when the detail is that Docker Desktop is not running, and for a missing context "The desktop-linux context is missing; open Docker Desktop once to create it, then Check Again.", either followed by "If you no longer use Docker Desktop, its folder is yours to delete in Finder." (deferred here from "VM images: Docker prune runs only while Docker Desktop answers"); "<reason>; Check Again when it ends." for `in-use`; "These items changed since the scan. Scan Again to see them as they are now."; the never cause in `reason()` words. The header: "frees about X; Y will be skipped", "frees about X before the check".
5. **The sheet.** `ConfirmSheet.swift`: Clean `.disabled(model.isCleaning || !sheet.cleanAllowed)`; the header; `warningsLine` uses the sentence and shows **Stop** while `.checking`.
6. **Tests.** New `SkipSummaryTests` (pure, from `ReviewFixture`): all owner-running; 2 of 6; each warnings state; changed-since-scan only; a mix. `CleanRunTests` with `FakeCLI`: a dry run of all `owner-running` → `cleanConfirmedItems()` runs no clean; Stop during a gated dry run → `.stopped`, clean allowed. `ContractTests.cleanHasOneOwner` extended to the `cleanAllowed` guard and the button's `!sheet.cleanAllowed`.

## Acceptance Criteria

- [ ] Test (DDCKit): with every item's prediction `wouldSkip(ownerRunning)`, `nothingToClean` is true, the warnings sentence names each app once in the order first seen, and `cleanConfirmedItems()` starts no CLI run (fake runner records no call).
- [ ] Test (DDCKit): with 2 of 6 items predicted to skip, the header's "frees about" equals the bytes of the other 4, and the skipped bytes are named.
- [ ] Test (DDCKit): while the check is `.checking`, the clean is not allowed; while `.waiting`, it is allowed and the header carries "before the check"; when the check then reports every item skipped, `nothingToClean` becomes true.
- [ ] Test (DDCKit): Stop during `.checking` ends the check as `.stopped`, and the clean is then allowed.
- [ ] Test (DDCKit): a check that `.failed`, was `.stopped` or is `.busy` leaves the clean allowed.
- [ ] Test (DDCKit): every item `wouldSkip(changedSinceScan)` → the line advises Scan Again, not Check Again; a mix of owner-running and changed-since-scan names both, owner apps first.
- [ ] Contract test (source check, `ContractTests` style): the sheet's Clean button is disabled on `nothingToClean`, and `cleanConfirmedItems()` guards it.
- [ ] Manual (owner): with Google Chrome running, select only the Chrome cache, open the sheet: the button is disabled and the line says to quit Google Chrome. Quit Chrome, Check Again: the button is enabled and the clean deletes it.

## Final Summary

## Dependencies

- "Review and clean window" (DDC-002): the confirmation sheet
- "Faster clean: re-check only what the selection needs" (DDC-002): the warnings check this story reads
- "Release 0.2.1: the Review window in the Dock, and skip what a running process holds open" (DDC-002): the `owner-running` skip
- The other stories from the same recording (DDC-002): "VM images: Docker prune runs only while Docker Desktop answers", "Review window: after a clean, only skips that can clear stay selected", "The Trash: its own row, not Your data", "User data: move a folder's picked files to the Trash from that folder", "Whole disk: free the checked areas in one sheet"

## Attachments

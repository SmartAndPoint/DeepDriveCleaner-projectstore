---
type: story
id: "story-windows-app-review-window-and-clean-with-confirmation"
epic: "DDC-008"
title: "Windows app: Review window and clean with confirmation"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [windows, tauri, app, review, clean, safety]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Windows app: Review window and clean with confirmation

| Field | Value |
|---|---|
| **Epic** | [DDC-008](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The Review window and the confirmation sheet of the macOS app, on Windows ([[adr/windows-app-tauri-2-shell-over-the-ddc-cli|Windows app: Tauri 2 shell over the ddc CLI]]). This is where a person removes what a scan found, so it carries the safety model unchanged: the CLI decides what may go, the app shows it and asks.

Parity with these DDC-002 stories, each a row in this story's parity table when it closes:
- "Review and clean window"; "Review window: sorted sidebar, Select all and folder groups";
- "Review window: clean only the selection in this group"; "Review window: Deselect All, then pick across groups and clean only that";
- "Review window: sort a group's rows by name, size or last change";
- "Review window: coding agents' data as a group"; "Review window: pick a coding agent's sessions one by one" (shown only where the CLI declares `coding-agents`);
- "Review window: after a clean, only skips that can clear stay selected"; "Review window: Clean is not offered when every item would be skipped";
- "Review window: say when the saved scan is not of the home folder";
- "HIG pass: keyboard, colour, empty states and menus": the filter field, Return and Escape;
- "User data: pick your own files for the Trash"; "User data: move a folder's picked files to the Trash from that folder"; "The Trash: its own row, not Your data" (later: the Recycle Bin's folder is outside the home, so it is a Whole disk area until the Windows catalog gains a per-user finding);
- "Protected by you: say what it is, and Remove Protection";
- Reveal in Explorer per row (`explorer /select,`);
- the confirmation sheet with its dry-run warnings, "Delete permanently" and "Delete all permanently" (0.14.10);
- the clean result with its skips, **Open the Recycle Bin**, **Show the Journal**, **Scan Again** and **Done** (`CleanResultView`).

**Windows differences**

- What goes where is the macOS rule: `review` items and picked user data go to the Recycle Bin, `safe` and `rebuildable` caches and build outputs are deleted; "Delete permanently" sends `review` items past the Recycle Bin. An item the Recycle Bin cannot take fails and is reported, never deleted in its place (app ADR §8).
- A file held open by another program is skipped and shown as "held open" (the Windows guard of the safety ADR, from story "Windows scanner and rule catalog"); the sheet's dry run names these before the clean where it can.
- "Remove Protection…" (`ddc keep remove`) and "Show the Journal" open the same places as on macOS; the journal opens in the default text editor.

## Decomposition

- [ ] `ddc-app-model`: `ReviewModel`, `ReviewItems`, `GroupLayout`, `CleanWording`, `CleanRun`, `CleanProgress`, `CleanResult`, `SkipSummary`, `RemoveProtection`, ported from `DDCKit` with their tests.
- [ ] The Review window: sidebar of groups, rows with sort headers, selection, the coding agents' sessions.
- [ ] The confirmation window: classes, sizes, dry-run warnings, the Recycle Bin or permanent choice; `clean --items <file> --json --progress json` only after the confirm (ADR §8).
- [ ] Progress and the result view; the panel and the tray state update after the clean.

## Implementation Plan

## Acceptance Criteria

- [ ] Every rule in the parity list has a `ddc-app-model` test, ported from the `DDCKit` test of the same rule.
- [ ] A test proves the app runs `clean` only with an item list the confirmation window produced, and never at start, on a schedule or from a notification.
- [ ] On Windows, a clean of fixture items against the real CLI moves the `review` items to the Recycle Bin and deletes the `rebuildable` ones, and a held-open file is reported skipped (CI, in the test home only).
- [ ] Manual (owner, lab machine, with the owner's go-ahead for the clean): Review shows the lab home's findings, a selection is cleaned to the Recycle Bin, and "Delete permanently" removes without the Recycle Bin.

## Dependencies

- Story "Windows app: Tauri shell, tray icon and the panel".
- DDC-007 story "Windows scanner and rule catalog" (the held-open guard, the Recycle Bin).

## Final Summary

## Attachments

-

---

*Last updated: 2026-11-01*

---
type: story
id: "story-whole-disk-in-the-app"
epic: "DDC-002"
title: "Whole disk in the app"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [app, ui, whole-disk]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Whole disk in the app

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

This story shows the report of "Whole disk: what lies outside the home folder" (DDC-002) in the app. It is read-only: no button here frees anything. The actions belong to "Whole disk: free space with the owning tool".

**Where it opens:**
- in the popover, a "Where does the rest go?" link under the totals;
- in the Review window, a "Whole disk" row at the bottom of the sidebar.

**What it shows:**
- **A bar of the startup container.** The home folder is drawn with what the last scan can free in the class colours, and its remainder in a neutral tone. The other areas and "not counted" are drawn in neutral tones too.
- **One row per area**, with:
  - its size, or "at least" for a partial area;
  - what the area is and what frees it;
  - for the home folder, the source and the date: "from the scan at 10:00", or "measured now".
- **"Not counted" is a row of its own**, with its explanation and the number of unreadable folders. An inconsistent report says so in the row and does not draw a negative segment.
- **The date of the whole-disk scan** and a "Scan whole disk" button, with progress and Stop. A stopped scan shows its partial rows as partial.

**How it runs:**
- `ddc disk` is read-only and takes no ddc lock. The app runs it outside the `CLIRunner` queue, in a lane of its own, so a scan of several minutes never delays Scan now, the scheduled scan or a clean.
- Only one whole-disk scan runs at a time.
- The app never starts it on its own. The scheduled scan stays the home folder's.

## Decomposition

Rough until the plan gate.

- [ ] **DDCKit:**
  - `DiskReport` decodes `last-disk.json` (schema version 1). A report of another version is not decoded loosely; the view shows that the app and the CLI do not match, as for the other reports (menu bar ADR §1).
  - A `DiskModel` holds the running state, Stop and the progress.
  - The `CLIRunner` gains a separate lane for `.disk` that does not wait on, or block, the queue.
- [ ] **App:**
  - the Whole disk view, used both in a window and as the Review window's detail;
  - the popover link;
  - the sidebar row.
- [ ] **Amendment to the menu bar ADR, §1**, before code: the runner serialises the runs that take the ddc lock. `ddc disk`, which takes none, runs beside them in a lane of its own, one at a time. `ddc disk free` takes the lock and stays in the queue ("Whole disk: free space with the owning tool").
- [ ] **Design:** an artboard on the design canvas before code, following the earlier stories.

## Implementation Plan

## Acceptance Criteria

- [ ] Test: `DiskReport` decodes a fixture of every area kind: partial, inconsistent, home from `last-scan`, home from `walk`, and no runtimes. A report of another schema version is not decoded and shows the mismatch.
- [ ] Test with a fake CLI: a `disk` run held at a gate does not delay a scan or a clean queued in the `CLIRunner`, and a second "Scan whole disk" joins the running one.
- [ ] Test: Stop sends SIGTERM, and the model then shows the cancelled report's partial rows.
- [ ] The §1 amendment is accepted after a critic pass.
- [ ] Test: nothing in the scheduled path runs `disk`. A contract test names the view's button as the one caller.
- [ ] Manual (owner): the view opens from the popover and from the Review window, reads correctly in light and dark, and its bar matches the rows.

## Final Summary

## Dependencies

- "Whole disk: what lies outside the home folder" (DDC-002): the report this view shows
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]:
  - §1, one runner;
  - this story amends §1 for a read-only lane beside the queue.

## Attachments

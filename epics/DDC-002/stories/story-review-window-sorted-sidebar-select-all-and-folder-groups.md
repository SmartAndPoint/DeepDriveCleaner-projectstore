---
type: story
id: "story-review-window-sorted-sidebar-select-all-and-folder-groups"
epic: "DDC-002"
title: "Review window: sorted sidebar, Select all and folder groups"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-20
updated: 2026-10-31
external_refs: {}
tags: [app, review-window, ux]
code_refs: []
specs: []
started_at: "2026-10-20T06:20:37.075Z"
closed_at: "2026-10-31T10:30:47.080Z"
plan_updated_at: "2026-10-20T06:20:37.075Z"
---

# Review window: sorted sidebar, Select all and folder groups

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | done |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner's feedback after the first try of "Review and clean window" (2026-10-20). It is drawn as board 9, "Review window, revised (2026-10-20)", of the design canvas, and the owner approved that board with its `tint` sidebar.

The window works, but it reads badly:
- the sidebar is a grey source list in the order of `rules --json`;
- a group repeats its rule, action and class on every row;
- a group with dozens of folders is one flat list.

This story changes how the Groups view is laid out and ordered. What can be selected, the confirmation sheet, the clean and the Items view stay as they are.

## Decomposition

- [x] **Sidebar on the window's own background**, not a grey source list.
  - Each class section (Safe, Rebuildable, Review) sits on a light tint of its class colour: about 7 % of `ReportClassStyle` in light mode, a matching tint in dark mode.
  - The selected entry gets a stronger tint (about 16 %) and a semibold name.
  - The section header is a darker shade of the class colour that passes 4.5:1 on its tint.
  - This is board 9's `tint`; its `glow` variant was not chosen.
- [x] **Sidebar order.** Within each class, entries are ordered by the group's total bytes, largest first; ties keep the `rules --json` group order. The order uses the total, not the selected amount, so an entry does not move while the user ticks.
- [x] **Select all.** The first row of a group's view is "Select all": a tri-state checkbox with the group's folder count and size. In a group with one rule it replaces the rule's header row.
- [x] **No repeated words on item rows.**
  - The rule id, the action, the class and "not selected by default" are said once: in the group's description, or on the rule's subgroup heading when the group has several rules. The recreate hint moves there too.
  - An item row keeps only what is its own: how long it has been untouched, why it is locked or will be skipped, and "frees ~" when that differs from its size.
- [x] **Subgroups, file-manager style.**
  - **A group with one rule:** its items fall into folder subgroups, one per first folder under the home folder (`~/temp`, `~/Projects`). Each item's path is shown relative to its subgroup.
    - Items carrying the report's `app-owned` note form one subgroup of their own, "Inside apps' own folders", with full paths.
    - A path outside the home folder falls under a subgroup named by its volume (`/Volumes/Work`), or by its first component elsewhere (`/opt`).
  - **A group with several rules:** one subgroup per rule, titled with the last part of its id (`npm` for `cache.npm`, because `rules --json` carries no display names), with the full id and the action beside it. A rule with one item is a single row, with no subgroup.
  - **The heading** is outdented and larger (15 pt semibold), with a folder icon on a folder subgroup, its count and its size. Its contents are indented. A chevron collapses it, and its checkbox is tri-state over its selectable rows.
  - **Order and state:**
    - subgroups are ordered by size, largest first, and items by size within each subgroup;
    - every subgroup is expanded when the group opens;
    - what is collapsed lasts while the window stays open.
- [x] **The group's header** shows "selected of total", as the sidebar does.
- [x] **Accessibility.**
  - The chevron is a button that reports expanded or collapsed and names its subgroup.
  - Every checkbox is labelled.
  - A tint never carries meaning alone: the class name heads each section.

## Implementation Plan

Planned with `projectstore:planner` (2026-10-20). Logic and wording go in DDCKit as pure code covered by unit tests; the views only draw them. There is no Rust change, and the Items view stays as it is.

**DDCKit**

1. **`ReviewItems.swift`**
   - Add `total(_ entry:)`, the sum over `rows(in:)`.
   - `sidebar` sorts each class's entries by that total, largest first, with ties broken by the index in `groupOrder`. It never reads the selection, so the order cannot move while the user ticks.
2. **New `GroupLayout.swift`.** The layout is pure and never reads the selection.
   - **Types:**
     - `Shape`: `folders` for one rule, `rules` for several;
     - `Key`: byte-exact path bytes or the full rule id, so `rust.target` and `maven.target` stay two subgroups, and NFC and NFD spellings of a name stay apart;
     - `Item`, `Subgroup` (kinds `folder`, `appOwned` and `rule`), `Single` and `Block`.
   - **`Line`** is `selectAll`, `heading`, `item` or `single`.
   - **`lines(collapsed:)`** returns the flat list the view draws.
   - **`ReviewItems.layout(_:matching:)`** builds a layout.
     - The shape, and whether a rule is a single row, are decided before the filter. The filter only removes items.
     - A note that every row under a heading carries is said once, on that heading.
3. **`CheckState`** moves from the app into DDCKit.
4. **`ReviewModel.swift`** gains:
   - `collapsed`, which lasts while the window is open;
   - `toggleCollapsed`, `windowClosed()`, `checkState(_:)` and `toggleAll(_:)`, which go through `set(_:selected:)`, so locked rows never change;
   - `amount(of:)`, the "selected of total" for the sidebar and the header.
5. **`CleanWording.swift`** gains `className`, `classBlurb` (reworded: the review blurb no longer says everything goes to the Trash), `hint`, `actionText` and `ownDetail`.
   - `ownDetail` keeps the lock reason, "Untouched N days", the activity notes, the notes left on the row and "incomplete".
   - `detail` stays byte-identical for the Items view, and a test pins it.
6. **`Formatting.swift`** gains `Wording.amount(selected:total:)` and `Wording.escaped`, taken out of `tilde`, whose output does not change.

**App**

7. **`ReportClassStyle`** (`PopoverView.swift`) gains `sectionTint`, `selectedTint` and `ink`, through `NSColor(name:)` with light and dark values.
   - Light: 7 % and 16 % of the class colour.
   - Dark: 14 % and 28 % of brighter bases.
   - The inks pass 4.5:1 on their tints.
8. **`ReviewView.swift`**
   - **`Sidebar` is rewritten:**
     - tinted sections on the window's background, with the class name as an `.isHeader`;
     - entries as buttons with `.isSelected`;
     - ↑ and ↓ through `onMoveCommand`.
   - **`GroupView` and `RuleHeader` go.**
   - **`ItemRow`** gains optional `shown` and `detail`. When they are `nil`, the Items view draws as it does today.
9. **New `ReviewGroupsView.swift`:**
   - the group header, then one flat `List` of lines with stable ids;
   - `SelectAllRow`;
   - `SubgroupHeading`, a chevron button with an expanded or collapsed value, a tri-state `CheckBox` and a hidden folder icon;
   - `SingleRow`.
10. **`ReviewWindowController.swift`:** a `willClose` observer calls `model.windowClosed()`.

**Tests**

New `ReviewGroupsTests.swift`, with builders in `ReviewSupport.swift`; one test or more per criterion.
- **Sidebar order**, including a tie and selection changes.
- **The folders shape:**
  - folders under `~` and under `/Volumes` and `/opt`;
  - the app-owned subgroup;
  - an item directly in `~`;
  - an NFC and NFD pair.
- **The rules shape:** titles from the id, single rows, and colliding titles.
- **Tri-state** with a scan-root lock.
- **Row detail** free of the id, the action, the class and "not selected by default", while the headings keep them; the pinned `detail`.
- **Collapse state.**

**Pitfalls**
- Selection stays keyed by the finding's index.
- The structural test scans flat folders and strips only whole-line comments, so new files stay flat and name no clean run kind.
- DDCKit stays free of AppKit and SwiftUI.
- The layout is built where the selection is not read, so ticking does not rebuild it.
- Fixtures need a size of at least 64 KiB, and must avoid the `cache.pip` rule, which the fixture's settings switch off.

**Seam for "User data: pick your own files for the Trash".** `Shape` and `Line` are enums that story extends with its own shape and line kinds. `selectAll: [Int]?` and `Subgroup.checkable` already express "no Select all, no heading checkbox", so the User data entry gets `selectAll = nil` already here.

## Acceptance Criteria

- [x] Unit test: sidebar entries within a class are ordered by total bytes, largest first, and the order does not change when the selection does — evidence: `ReviewGroupsTests.sidebarOrder`
- [x] Unit test: subgroups follow the decomposition and are ordered by size, as are the items in each:
  - a one-rule group's subgroups are the first folders under home, with `app-owned` items in a subgroup of their own and paths relative to their subgroup;
  - a several-rule group has one subgroup per rule, titled by the id's last part;
  - a one-item rule is a single row.

  Evidence: `ReviewGroupsTests.foldersShape`, `.rulesShape`.
- [x] Unit test: Select all and a subgroup heading are off, mixed or on over their selectable rows; toggling one changes only its selectable rows; locked rows never change — evidence: `ReviewGroupsTests.triState`
- [x] Unit test: an item row's detail line holds none of the rule id, the action, the class or "not selected by default" — evidence: `ReviewGroupsTests.ownDetail`
- [ ] Manual (owner, moved to the owner-checks runbook at close, 2026-10-31):
  - the sidebar tints in light and dark mode;
  - collapsing and expanding;
  - Select all and a heading's checkbox;
  - a group of 500 items opens and scrolls without visible lag, and a subgroup collapses at once.

## Final Summary

**What changed.** The Review window's Groups view, re-laid out after the owner's first try (board 9, `tint`). Shipped in 0.2.0.
- The sidebar sits on the window's background, each class section on a light tint of its colour, ordered by each group's total bytes; ticking never moves an entry.
- A group's view starts with a tri-state Select all row; item rows keep only what is their own, and the rule, action, class and default note move to the group's description or the subgroup heading.
- A one-rule group falls into folder subgroups (the first folder under home, a volume, `app-owned` apart, paths relative); a several-rule group has one subgroup per rule. Headings fold, carry a tri-state checkbox, and keep their state while the window is open.
- The group's header and the sidebar say "selected of total"; the chevron and every checkbox carry accessibility labels and values.

**Why.** The owner's feedback of 2026-10-20 on "Review and clean window" (DDC-002): a grey sidebar in catalog order, rows that repeat their rule, and flat lists of dozens of folders.

**Tests.** `ReviewGroupsTests` as above (also `.collapse`, `.amount`, `.userDataNoSelectAll`), run by the Gate's `app` job (green on main at 2cdae98, 2026-10-31). Closed under the owner's rule of 2026-10-30: the manual check moved to the runbook "Owner checks: manual acceptance across stories".

**Risks and follow-ups.**
- The tints in light and dark mode and the speed of a 500-item group are manual checks in the runbook; no test measures the scrolling.

## Technical Notes

- The grouping is pure: DDCKit computes the subgroups and rows from `ReviewItems`, so every case is a unit test, and the view only draws them.
- The `app-owned` note comes from the core (`rules/classify.rs`, `app_owned`) through the report's `notes`; `CleanWording.note` already words it.
- Design: board 9 of the design canvas (https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV); the board's sizes are illustrative.

## Dependencies

- "Review and clean window" (this story reworks its Groups view)

## Attachments

- Design canvas, board 9: https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV

---

*Last updated: 2026-10-31*

---
type: story
id: "story-review-window-sort-a-groups-rows-by-name-size-or-last-change"
epic: "DDC-002"
title: "Review window: sort a group's rows by name, size or last change"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-29
updated: 2026-10-29
external_refs: {}
tags: [core, app, review, sorting, user-report]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Review window: sort a group's rows by name, size or last change

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

Requested by the owner on 2026-10-29, with a screenshot of the Python environments group (app 0.4.0): "it lacks sorting by date … add separate columns: files and paths, size, date of last use (change)".

What the screenshot shows. A folder subgroup ("<folder> · 15 folders") lists `.venv` folders, largest first. That is the only order the Groups view has (`GroupLayout.ordered`: bytes, then id). Each row has its path, a second line such as "Untouched 33 days · No lock file", and its size with "frees ~…" under it. To find the environments nobody has used in years, the user has to read every second line.

**What exists today.** The Items view (the flat list of every finding except user data) has a sort Picker: Biggest First, By Path, Most Idle First (`ReviewItems.Sort`, held in `ReviewModel.sort`, `ReviewView.swift`). The Groups view has no sort control. "Most Idle First" and the "Untouched N days" text both come from the *project's* activity (`Project.idle_days`, read from git, manifests and lock files). That covers only findings inside a project. It is missing for an active project, and the cache groups have no date at all: npm, Homebrew, Docker, Xcode DerivedData and the rest. Neither tells when the folder itself last changed.

**Why a new date, not a sort on the data already there.** A Groups-view sort over size, name and project activity would need no core change. The owner asked for the date of last use or change of the item itself, though, and for most groups no such date exists today. The new field is what makes the Changed column answer the request in every group. It also makes the sort mean the same thing in a project group and in a cache group.

**Three columns, each sortable, one sort state.** In the Groups view and in the Items view, the rows get a header with three columns:

- **Name**: the path as today, with its second line.
- **Size**: bytes, with "frees ~…" under it as today.
- **Changed**: see *The date* below.

Clicking a column title sorts by it. Clicking it again reverses the order, and the header shows an arrow on the sorted column. The default is Size, largest first. That is today's order in both views, so nothing moves for a user who never clicks.

- **The header replaces the Items-view Picker.** `ReviewModel.sort` becomes the one sort state, shared by both views. Its type is `ReviewSort` (column and direction), which replaces `ReviewItems.Sort`. "Biggest First" is Size descending and "By Path" is Name ascending. "Most Idle First" is dropped as an order: Changed, oldest first, answers the same question for every finding, and the project's activity stays on the second line.
- **Name** sorts by the path as shown, case-insensitively, with numbers in numeric order (Finder's order). This is a small visible change from today's "By Path", which sorts by bytes.
- **The Items view's heading** "Largest items" names the sort instead (for example "Items · oldest change first").
- **Changed** sorts oldest first on the first click, because the oldest is what the user looks for. Rows with no date sort last in either direction, and so does a subgroup with no dated row. Ties fall back to size, then id.
- **Subgroups follow the sort.** It applies within each folder subgroup and to the subgroups themselves: by their summed size, their newest date, or their title. A subgroup heading shows its newest date in the Changed column, so the order can be explained. A `.single` row is a row like any other. The sidebar's order does not change (story "Review window: sorted sidebar, Select all and folder groups").
- **The filter comes first.** The sort orders the rows that pass it (`matches`).
- **Where the header sits.** It sits under the group's header and the Select all row, above the first row. There is no SwiftUI `Table`: the rows stay `ItemRow` stacks in the `List`, with fixed-width, right-aligned Size and Changed columns. Each column title is a button. Its accessibility label reads like "Changed, sorted oldest first", and it can be reached and pressed from the keyboard (the bar set by "HIG pass: keyboard, colour, empty states and menus").
- **The sort is kept** across groups and launches through `AppPreferences` (a new `ReviewSortKey`, a per-user convenience). A missing or unreadable value means the default.
- **Sorting never changes** the selection, the Select all row, or what a group's Clean button cleans.

**The date.**

- **A finding** shows the newest modification time (`mtime`) of anything inside it, its own directory included. For a `.venv` that is the last install into it, or the last `__pycache__` write when it was used. The tooltip says "Newest change inside: <date and time>". For an `incomplete` finding it says "at least" (the walk did not see everything).
- **A user-data entry** (Downloads and the other folders whose files can be picked) already has `FolderEntry.modified`: the entry's *own* `mtime`, which sees only changes to its direct children (story "User data: pick your own files for the Trash"). Its tooltip says "Changed: <date and time>". The entry's second line drops "Last changed N days ago", because the column now shows it.
- **Shown as** "Today", "Yesterday", or a date in the system's short style ("Sep 25, 2026"). These are relative to the scan's time (`report.generatedAt`, as the entry line measures days today), not to the clock. A row with no date shows "—". A time after the scan (a wrong clock) is shown as it is and sorts as the newest.
- The modification time is used, not the access time: macOS does not keep access times reliably.
- The second line's "Untouched N days" stays. It is the project's activity, a different signal. The planner decides at the work-start gate whether to rename it "Project untouched N days" so the two cannot be read as one.

**The core.** A finding carries no date today. The scan reads modification times only for a user-data folder's own listing (`DirReader::read_with_times`, which also asks for the added time).

- **The field.** `Finding` gains `modified: Option<String>`, RFC 3339: the newest `mtime` in its subtree. The walk aggregates it under the finding's claim, the same way as the sizes. An optional field keeps the scan schema at version 1 (`model.rs`). `docs/scan.schema.json` is regenerated (`DDC_UPDATE_SCHEMA=1`, which `report.rs` enforces). DDCKit's strict `ScanReport.Finding` decodes it as optional.
- **Only inside claims.** Times are asked for only when a directory is listed under a finding's claim. Regions get no `modified`, and neither does the user-data folder's own finding (`OwnerKind::Region`); their entries keep `FolderEntry.modified`. So the cost is measured on claimed trees only.
- **Reading the time.** The walk asks for `ATTR_CMN_MODTIME` only, not the added time. It goes into the `getattrlistbulk` request the walk already makes for the type, flags, file id, link count and allocated size (`sys/bulk.rs`): 16 bytes more per record and no extra system call. `StdReader` gets it from the `lstat` it already does. The claim root's own `mtime` comes from the stat both readers already make when they open a directory (`fstat` in `bulk.rs`, `symlink_metadata` in `reader.rs`). `DirListing` gains the directory's own `modified`, so it costs no extra system call. A root that cannot be opened has no listing, so it has no date.
- **After a clean.** The post-clean edit (`report/after_clean.rs`) keeps the date. A refreshed item takes the re-check's finding, date included. A tool-cleaned item gets the date from its re-measure. `Measure` gains it as seconds (`Option<i64>`, as `EntryMeta.modified` is), so it stays `Copy`. The RFC 3339 text is made at the report's edge.
- **The goldens.** Loadgen writes the build time as the `mtime` of every file except the old ones (`ddc-loadgen` `build.rs`). So `golden_cli`'s normalizer maps `modified` to `<build>` unless it is OLD, the same way it treats `last_activity`. The goldens are then re-recorded with `make golden-record`.
- **The cost.** It is expected to be negligible, and is measured to confirm. A `WalkOptions` field (on by default) turns the times off. It is reached through a hidden `ddc scan --no-times` flag, in the same way as `clean`'s hidden `--local-snapshots`. The protocol: build the `mixed` profile at size M on the load image, then run the release CLI's `scan --json` on it directly, 5 warm runs each way, and compare the median `duration_ms`. If times add more than 5 % to the median, the core part stops and the measurement goes to the owner before any other choice is made. There is no top-level fallback: for a `.venv` it would show the creation date, because pip writes three levels down. The numbers go into the Implementation Plan.

**For the owner at the work-start gate.** These are product changes the story makes, stated here so they can be confirmed:

- "Most Idle First" is dropped. It also put findings outside a project last, which Changed does not reproduce.
- Changed sorts oldest first on the first click, the opposite of Finder's Date Modified.
- "By Path" becomes Finder's order.

## Decomposition

- [ ] Core, part 1: MODTIME in the bulk request inside claims (split from the added time), `StdReader`'s `lstat` time, the root's `modified` on `DirListing`, the `WalkOptions` toggle and the hidden `scan --no-times`.
- [ ] Measure (gate): `mixed` at M, 5 warm runs with and without times, median `duration_ms`; record the numbers. Over 5 %: stop and go to the owner.
- [ ] Core, part 2: `Finding.modified` aggregated in the walk; `Measure` (seconds) and `after_clean` keep it; the schema regenerated; the golden normalizer and `make golden-record`.
- [ ] DDCKit: `ScanReport.Finding.modified` (optional); `ReviewSort` (column and direction) in place of `ReviewItems.Sort`; one ordering function used by `GroupLayout` for items and subgroups and by `items(matching:sort:)`; the subgroup's newest date; `CleanWording` for the Changed text and the tooltips, with the scan time passed in; the entry line without "Last changed"; `AppPreferences.reviewSortKey`.
- [ ] App: the column header in both views, which replaces the Items-view Picker, with the arrows, the accessibility labels and keyboard access; the Changed column in rows and subgroup headings.

## Implementation Plan

## Acceptance Criteria

- [ ] Test (core): a scan of a temp HOME with a `.venv` whose newest file sits three levels down with a set `mtime` reports that time as the finding's `modified`; a finding whose root cannot be opened has none; a region and the user-data folder's own finding have none.
- [ ] Test (core): the scan schema keeps version 1; the schema file has the optional `modified`; a report without it still decodes in DDCKit.
- [ ] Test (core): after a clean that skips an item, the saved report holds the re-check's `modified` for it; after a tool clean, the item holds its re-measure's `modified`.
- [ ] Test (core): the goldens pass with `modified` normalized, and an OLD file's time is kept.
- [ ] Measurement recorded in the Implementation Plan: the median walk time with and without the times, under the protocol above.
- [ ] Test (DDCKit): each column in each direction orders a fixture's items and subgroups as described, after the filter: Name in Finder's order; Size with ties by id; Changed oldest first, with undated rows and undated subgroups last both ways and ties by size, then id; the Items heading names the sort.
- [ ] Test (DDCKit): the default sort is Size, largest first, and gives today's order in both `GroupLayout` and the Items view; an unreadable stored sort reads as the default; the old "Biggest First" and "By Path" orders are Size descending and Name ascending.
- [ ] Test (DDCKit): sorting leaves the selection, the Select all state and a group's clean scope unchanged.
- [ ] Test (DDCKit): the Changed wording against a given scan time: "Today", "Yesterday", a short date, "—" with no date; the finding and entry tooltips; "at least" for an incomplete finding.
- [ ] Manual (owner): in Python environments, click Changed: the oldest environments come first; click again: the newest; in Items the same header works; quit and reopen: the sort is kept.

## Final Summary

## Dependencies

- "Review window: sorted sidebar, Select all and folder groups" (DDC-002): the folder subgroups and their order
- "User data: pick your own files for the Trash" (DDC-002): `FolderEntry.modified`
- "Review window: after a clean, only skips that can clear stay selected" (DDC-002): the post-clean edit that keeps a refreshed finding
- "HIG pass: keyboard, colour, empty states and menus" (DDC-002): the keyboard and VoiceOver bar for the header
- DDC-005: the load image and its `mixed` profile for the measurement

## Attachments

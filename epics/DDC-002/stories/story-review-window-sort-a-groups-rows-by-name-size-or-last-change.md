---
type: story
id: "story-review-window-sort-a-groups-rows-by-name-size-or-last-change"
epic: "DDC-002"
title: "Review window: sort a group's rows by name, size or last change"
status: in-progress
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-29
updated: 2026-10-30
external_refs: {}
tags: [core, app, review, sorting, user-report]
code_refs: []
specs: []
started_at: "2026-10-30T14:46:34.935Z"
closed_at: null
plan_updated_at: "2026-10-30T14:46:34.935Z"
---

# Review window: sort a group's rows by name, size or last change

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
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
- **Name** sorts by the path as shown, case-insensitively, with numbers in numeric order (Finder's order). This is a small visible change from today's "By Path", which sorts by code point, case-sensitively.
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
- **Shown as** "Today", "Yesterday", or an abbreviated date ("Sep 25, 2026", `Date.FormatStyle(date: .abbreviated)`, with the calendar, locale and time zone injectable for tests). These are relative to the scan's time (`report.generatedAt`, as the entry line measures days today), not to the clock. A row with no date shows "—". A time after the scan (a wrong clock) is shown as it is and sorts as the newest.
- The modification time is used, not the access time: macOS does not keep access times reliably. So Changed is the last write, not the last use: Python writes `__pycache__` only on the first import after a change, and a folder opened in Finder gets a `.DS_Store`. The tooltip says "Newest change inside" for that reason, and "Project untouched" stays beside it.
- The second line's "Untouched N days" stays, renamed "Project untouched N days" (decided at the work-start gate). It is the project's activity, a different signal, and the new name keeps the two from being read as one.

**The core.** A finding carries no date today. The scan reads modification times only for a user-data folder's own listing (`DirReader::read_with_times`, which also asks for the added time).

- **The field.** `Finding` gains `modified: Option<String>`, RFC 3339: the newest `mtime` in its subtree. The walk folds every error-free listed entry's `mtime` into the owner's maximum before the match on its kind, so before the hard-link branch too (a file with `nlink > 1` goes to the inode table for its bytes, but its time still counts: pnpm and uv trees are mostly hard links), together with each directory's own `DirListing.modified`. An optional field keeps the scan schema at version 2 (`model.rs`). `docs/scan.schema.json` is regenerated (`DDC_UPDATE_SCHEMA=1`, which `report.rs` enforces). DDCKit's strict `ScanReport.Finding` decodes it as optional.
- **Only inside claims.** Times are asked for only when a directory is listed under a finding's claim. `Visitor::wants_times` defaults to false; `RuleVisitor` opts in, except for a `kept` claim and an unlisted rule (an app bundle), which show no date. The re-check's `Pruned` and the post-clean `Alone` visitors delegate it, so a refreshed finding keeps its date; `AgentsVisitor`, `disk`, `free` and the executor's walks do not ask. Regions get no `modified`, and neither does any `user-data` finding: the walk asks for times by the rule's class, as `RuleVisitor::wants_private` does, because Pictures, Movies and Music are known-path claims, not regions. Their entries keep `FolderEntry.modified`, and `--no-times` never switches off a user-data listing's own times, which decide its recent-file locks. So the cost is measured on claimed trees only.
- **Reading the time.** The walk asks for `ATTR_CMN_MODTIME` only, not the added time. It goes into the `getattrlistbulk` request the walk already makes for the type, flags, file id, link count and allocated size (`platform/macos/bulk.rs`): 16 bytes more per record and no extra system call. The exception is a finding claimed after its listing (`Post::Claim`, such as a `.venv` found by its `pyvenv.cfg`): its root was listed without times, so it is listed again once. Today the walker re-reads there only for private sizes (`walker.rs`, `!want_private && claim.want_private`), which a pip `.venv`, `target/` and npm `node_modules` never ask for; the condition widens to "or the claim wants times and the listing had none". A re-read for private sizes keeps today's error accounting. One for times only that fails leaves the date a lower bound: the finding is not marked `incomplete` for it, so its tooltip does not say "at least" (accepted: the root's own files are missing, never the subtree). `StdReader` gets it from the `lstat` it already does. The claim root's own `mtime` comes from the stat both readers already make when they open a directory (`fstat` in `bulk.rs`, `symlink_metadata` in `reader.rs`). `DirListing` gains the directory's own `modified`, so it costs no extra system call. A root that cannot be opened has no listing, so it has no date.
- **After a clean.** The post-clean edit (`report/after_clean.rs`) keeps the date. A refreshed item takes the re-check's finding, date included. A tool-cleaned item gets the date from its re-measure. `Measure` gains it as seconds (`Option<i64>`, as `EntryMeta.modified` is), so it stays `Copy`. The RFC 3339 text is made at the report's edge.
- **The goldens.** Loadgen sets every entry, folders included, to OLD (2024-01-01) unless it is marked fresh, such as `.git/index` and `package.json` (`ddc-loadgen` `desc.rs`, `platform/unix.rs::touch`). So `golden_cli`'s normalizer maps `modified` to `<build>` unless it is OLD, the same way it treats `last_activity`, and most goldens show the literal OLD date. The goldens are then re-recorded with `make golden-record`.
- **The cost.** It is expected to be negligible, and is measured to confirm. A `WalkOptions` field (on by default) turns the times off. It is reached through a hidden `ddc scan --no-times` flag, in the same way as `clean`'s hidden `--local-snapshots`. The protocol: build the `mixed` profile at size M on the load image, then run the release CLI's `scan --json` on it directly, built from the slice-1 branch before that PR merges. First a noise floor: 9 runs without times interleaved with 9 more without times, the first of each discarded. Then 9 with and 9 without, interleaved, the first of each discarded; the machine's load average is recorded with them. If the median difference exceeds the floor by more than 5 % of the median, the core part stops and the measurement goes to the owner before anything merges. There is no top-level fallback: for a `.venv` it would show the creation date, because pip writes three levels down. The numbers go into the Implementation Plan.

**For the owner at the work-start gate.** These are product changes the story makes, stated here so they can be confirmed:

- "Most Idle First" is dropped. It also put findings outside a project last, which Changed does not reproduce.
- Changed sorts oldest first on the first click, the opposite of Finder's Date Modified.
- "By Path" becomes Finder's order.

**Confirmed by the owner on 2026-10-30**: all three, as written.

## Decomposition

- [ ] Core, part 1: MODTIME in the bulk request inside claims (split from the added time), `StdReader`'s `lstat` time, the root's `modified` on `DirListing`, the `WalkOptions` toggle and the hidden `scan --no-times`.
- [ ] Measure (gate, before slice 1 merges): `mixed` at M under the protocol in *The cost*; record the noise floor, the medians and the load average. Over the bar: stop and go to the owner.
- [ ] Core, part 2: `Finding.modified` aggregated in the walk; `Measure` (seconds) and `after_clean` keep it; the schema regenerated; the golden normalizer and `make golden-record`.
- [ ] DDCKit: `ScanReport.Finding.modified` (optional); `ReviewSort` (column and direction) in place of `ReviewItems.Sort`; one ordering function used by `GroupLayout` for items and subgroups and by `items(matching:sort:)`; the subgroup's newest date; `CleanWording` for the Changed text and the tooltips, with the scan time passed in; the entry line without "Last changed"; `AppPreferences.reviewSortKey`.
- [ ] App: the column header in both views, which replaces the Items-view Picker, with the arrows, the accessibility labels and keyboard access; the Changed column in rows and subgroup headings.

## Implementation Plan

Planned 2026-10-30 against `main` at eea7d3c, with the planner's placement. Five slices in four PRs; DDCKit and the app ship together, as slices 4 and 5.

1. **Core, part 1: times in the listing** (`crates/ddc-core`).
   - `platform/types.rs`: a `Times` enum (`None`, `Modified`, `ModifiedAndAdded`) in place of the two read methods' split; `DirListing.modified: Option<i64>` (seconds) for the directory itself, from the open's `fstat` / `symlink_metadata`.
   - `platform/macos/bulk.rs`: `ATTR_CMN_MODTIME` joins the attribute mask when the listing asks for times; the added time only for `ModifiedAndAdded`. Record parsing in mask order, with the existing short-record checks.
   - `scan/reader.rs`: one read method with a `times` parameter; `StdReader` takes the time from its `lstat`.
   - The walk (`scan/walker.rs`, `scan/mod.rs`): `Visitor::wants_times(dir, tag)`, default false; `RuleVisitor` returns true for a listed rule that is neither `UserData` nor the `kept` tag; `WalkOptions.claim_times` (default on); `OwnerResult.modified`, the maximum of every error-free listed entry's `mtime`, folded before the match on its kind (so a skipped or unopenable subdirectory still counts its own), and of each directory's `DirListing.modified`. The `Post::Claim` re-read runs when `(!want_private && claim.want_private) || (!had_times && claim.want_times)`.
   - `ddc scan --no-times` (hidden), which sets only `claim_times`; user-data listings keep their own times.
   - `report/recheck.rs` (`Pruned`) and `report/after_clean.rs` (`Alone`) delegate `wants_times`; `free/measure.rs`'s `Alone` does not. `remeasure()` for a file fills `Measure.modified` from its metadata (slice 3).
   - Tests: a temp-HOME `.venv` with a deep newest file (`File::set_modified`), a `Post::Claim` finding whose newest file is a direct child of its root (`node_modules/.package-lock.json`), a hard-linked newest file, an unopenable root, a region and Downloads without a date; a reader test that the record layout parses with and without MODTIME. Slice 1 changes no output (`Finding.modified` arrives in slice 3), so these assert through `OwnerResult`.
2. **Measurement gate, before slice 1 merges** (main session, unsandboxed; never a subagent: it needs `hdiutil`). The DDC-005 load image, `ddc-loadgen build --profile mixed --size M` with a fake `HOME` (`env -i HOME=<fake>`), then the slice-1 branch's release CLI `scan --json --data-dir <tmp>` under the protocol above (noise floor, then 9 interleaved runs each way, first discarded, load average recorded); the medians of `duration_ms` go into this plan. Over the bar: stop, go to the owner, and slice 1 does not merge.

   **Measured 2026-10-30** on the slice-1 branch (`walk-times`, 99afd44), release build. The image is an APFS sparse bundle built as `scripts/load-image.sh` builds the DDC-005 one (`.metadata_never_index`, no fseventsd log), filled by `ddc-loadgen build mixed --entries 100000 --seed 1` (size M). Each run is `env -i HOME=<fake> PATH=/usr/bin:/bin deepdrivecleaner scan --json --data-dir <tmp> [--no-times] <tree>`. There were 10 interleaved runs per arm and the first of each was discarded, so 9 count. Wall is the report's `duration_ms`, the scan alone. CPU is `/usr/bin/time` user + sys for the whole process, the JSON write included, which is the same in both arms. The floor is the A/A difference over the smaller A/A median. The difference is times against `--no-times`, over the `--no-times` median.

   | Pair | Median wall | Median CPU |
   |---|---|---|
   | A/A, both `--no-times` | 1519 / 1482 ms, floor 2.5 % | 10880 / 10720 ms, floor 1.5 % |
   | times / `--no-times` | 1492 / 1449 ms, +3.0 % | 11320 / 10770 ms, +5.1 % |

   The 1-minute load average was between 6.4 and 9.0 on 10 cores, read at every run. Wall runs spread 11–13 % within a series (min to max over the median). CPU runs spread 25–30 %, in two regimes as the load rose; the paired per-run difference, which that drift does not bias, is +250 ms (+2.3 %) at the median, positive in 7 of 9 pairs. The excess over the floor is +0.5 % wall and +3.6 % CPU, both under the 5 % bar: slice 1 passes. The CPU figure is the closer call. Its expected source is 16 bytes more per `getattrlistbulk` record plus the widened `Post::Claim` re-read, and it is within the bar.

   Two earlier runs that day, under a load of 18–27 from another session's builds, also passed. Their A/A arms ran with times rather than without, and the per-run load was not kept. They are superseded by the run above: one gave −2.3 % wall against a 2.9 % floor and +0.2 % CPU against 3.4 %; the other gave +3.0 % wall against 8.2 %.
3. **Core, part 2: the report.** `Finding.modified: Option<String>` (RFC 3339 by a `rfc3339_secs` helper next to the existing time formatting, fixed-width UTC, so the app can compare the strings); the `Finding { … }` literals in `plan.rs`, `exec/preflight.rs`, `ddc-cli/src/prompt.rs` and `ddc-loadgen/tests/properties.rs`; `Measure.modified: Option<i64>` and `report/after_clean.rs` keeping it; `docs/scan.schema.json` regenerated (`DDC_UPDATE_SCHEMA=1`), schema version stays 2; `golden_cli`'s normalizer maps a non-OLD `modified` to `<build>`; `make golden-record`.
4. **DDCKit and the app, one PR** (slice 4 alone would leave the app target, which `make gate` does not compile, broken on `main` until slice 5). DDCKit (`app/Packages/DDCKit`): `ScanReport.Finding.modified` optional, and `Finding.init(entry:folder:)` sets it from `entry.modified`, so one comparator serves both; `ReviewSort { column, ascending }` replacing `ReviewItems.Sort`, with one comparator shared by `GroupLayout` (rows and subgroups) and `items(matching:sort:)`; Name by `localizedStandardCompare`; Changed with undated last both ways, ties by size, then id; the subgroup's newest date; `CleanWording` for the Changed text, tooltips and "Project untouched N days", with the scan time, calendar, locale and time zone passed in; the entry line without "Last changed"; `CleanWording.itemsHeading(_:)` for the Items heading; `AppPreferences.reviewSortKey` with a tolerant decode; the `Subgroup.items` and `Block` doc comments that say "largest first".
5. **App, in the same PR** (`app/DeepDriveCleaner`): a `SortHeader` view under each group's Select all row and above the Items list, replacing the Picker; buttons with arrows and accessibility labels; fixed-width right-aligned Size and Changed columns in `ItemRow` and the subgroup headings; the Items heading from `CleanWording`. Then a release.

Critic round 1 (2026-10-30) gave the widened re-read, `wants_times`' default and delegation, the hard-link fold, the one PR for DDCKit and the app, the measurement before the merge and its noise floor, and the heading in DDCKit.

Footprint (proposed `code_refs`): `crates/ddc-core/src/platform/types.rs`, `crates/ddc-core/src/platform/macos/bulk.rs`, `crates/ddc-core/src/scan/reader.rs`, `crates/ddc-core/src/scan/walker.rs`, `crates/ddc-core/src/scan/mod.rs`, `crates/ddc-core/src/rules/visitor.rs`, `crates/ddc-core/src/model.rs`, `crates/ddc-core/src/report/after_clean.rs`, `crates/ddc-core/src/report/recheck.rs`, `docs/scan.schema.json`, `crates/ddc-cli/tests/golden_cli.rs`, `bench/golden/`, `crates/ddc-cli` (the scan flag), DDCKit's `ScanReport.swift`, `ReviewItems.swift`, `GroupLayout.swift`, `CleanWording.swift`, `ReviewModel.swift` and `AppPreferences.swift`, the app's `ReviewView.swift`, `ReviewGroupsView.swift` and a new `SortHeader.swift`.

## Acceptance Criteria

- [ ] Test (core): a scan of a temp HOME with a `.venv` whose newest file sits three levels down with a set `mtime` reports that time as the finding's `modified`; a finding whose root cannot be opened has none; a region and every user-data finding (Downloads, Pictures) have none.
- [ ] Test (core): a `Post::Claim` finding whose newest file is a direct child of its root, and one whose newest file is hard-linked, report that file's time; `scan --no-times` gives no finding a `modified`, while a Downloads entry keeps its own.
- [ ] Test (core): the scan schema keeps version 2; the schema file has the optional `modified`.
- [ ] Test (DDCKit): a report without `modified` still decodes.
- [ ] Test (core): after a clean that skips an item, the saved report holds the re-check's `modified` for it; after a tool clean, the item holds its re-measure's `modified`.
- [ ] Test (core): the goldens pass with `modified` normalized, and an OLD file's time is kept.
- [ ] Measurement recorded in the Implementation Plan: the noise floor, the median walk time with and without the times, and the load average, under the protocol above.
- [ ] Test (DDCKit): each column in each direction orders a fixture's items and subgroups as described, after the filter: Name in Finder's order; Size with ties by id; Changed oldest first, with undated rows and undated subgroups last both ways and ties by size, then id; the Items heading (`CleanWording.itemsHeading`) names the sort.
- [ ] Test (DDCKit): the default sort is Size, largest first, and gives today's order in both `GroupLayout` and the Items view; an unreadable stored sort reads as the default; the old "Biggest First" and "By Path" orders are Size descending and Name ascending.
- [ ] Test (DDCKit): sorting leaves the selection, the Select all state and a group's clean scope unchanged.
- [ ] Test (DDCKit): the Changed wording against a given scan time: "Today", "Yesterday", an abbreviated date, "—" with no date; the second line says "Project untouched N days"; the finding and entry tooltips; "at least" for an incomplete finding.
- [ ] Manual (owner, to the owner-checks runbook at close): in Python environments, click Changed: the oldest environments come first; click again: the newest; in Items the same header works; quit and reopen: the sort is kept.

## Final Summary

## Dependencies

- "Review window: sorted sidebar, Select all and folder groups" (DDC-002): the folder subgroups and their order
- "User data: pick your own files for the Trash" (DDC-002): `FolderEntry.modified`
- "Review window: after a clean, only skips that can clear stay selected" (DDC-002): the post-clean edit that keeps a refreshed finding
- "HIG pass: keyboard, colour, empty states and menus" (DDC-002): the keyboard and VoiceOver bar for the header
- DDC-005: the load image and its `mixed` profile for the measurement

## Attachments

---
type: story
id: "story-user-data-pick-your-own-files-for-the-trash"
epic: "DDC-002"
title: "User data: pick your own files for the Trash"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-20
updated: 2026-10-31
external_refs: {}
tags: [cli, app, safety, user-data]
code_refs: []
specs: []
started_at: "2026-10-20T07:19:01.485Z"
closed_at: "2026-10-31T10:30:47.181Z"
plan_updated_at: "2026-10-20T07:19:01.485Z"
---

# User data: pick your own files for the Trash

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | done |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner's decision of 2026-10-20: the Review window opens User data, and the user can pick their own files and move them to the Trash.

Until now user data never entered a plan: the safety ADR, §1, gives it the `none` method, and the executor refuses it. This story lets hand-picked entries inside five user-data folders into a plan, to the Trash only, and keeps the rest of user data out. The owner chose this over two alternatives:
- browsing only, with deletion left to Finder;
- treating user data as ordinary `review` items, permanent deletion included.

The rules are in the safety ADR's amendment of 2026-10-20, and board 9 of the design canvas draws the screen.

**Terms.** An **entry** is a direct child of one of the five folders (for iPhone and iPad backups, one backup folder). The **folder** is the user-data finding it lies in.

## Decomposition

**CLI: what the scan lists**

- [x] **`scan` lists each folder's entries.**
  - It covers each finding of `user.downloads` (the remainder), `user.pictures`, `user.movies`, `user.music` and `user.ios-backups`.
  - `user.imazing-backups` stays report only until its backup layout is checked on a Mac with iMazing: without that, the fail-closed `Status.plist` check below could lock every iMazing backup for good.
  - The report lists the finding's entries as an optional `contents` list.
  - Each entry carries its path, bytes, `dev` and `ino`, kind (`file` or `folder`), modification time, and a lock reason when it has one.
  - Entries come largest first: at most 100, each at least 10 MiB, and the rest as a count and a byte total.
  - Only regular files and directories are listed: never a symlink or another kind, and never a dataless (`SF_DATALESS`) entry.
  - An entry whose own metadata cannot be read is not listed. An entry with parts it may not read is listed with "partly unreadable", its size a lower bound, and stays pickable: the Trash keeps it.
  - The field is additive. `SCHEMA_VERSION` of the scan report stays: a new optional field keeps it (`model.rs`), and the Swift decoder ignores unknown keys (verified). `LISTED_SCHEMA_VERSION` of the item list stays as well, because the app and the CLI ship together.
- [x] **Locked entries** are listed with their reason, and can never be picked.
  - **Managed by an app**, from a maintained list, named by the app:
    - library packages by extension: `.photoslibrary`, `.photolibrary`, `.musiclibrary`, `.tvlibrary`, `.imovielibrary`, `.fcpbundle`, `.lrcat`, `.lrcat-data`, `.lrcat-wal`, `.lrdata`, `.lrlibrary` (Lightroom), `.aplibrary` (Aperture), `.cocatalog`, `.cosessiondb` (Capture One);
    - fixed paths: `Music/Music`, `Music/iTunes` (a library upgraded from older macOS keeps its media there), `Music/GarageBand`, `Music/Audio Music Apps`, `Movies/TV`, `Movies/Motion Templates.localized`, `Movies/Final Cut Backups.localized`, `Pictures/Photo Booth Library` and `Pictures/Lightroom` (Lightroom Classic keeps its catalog one level down, so the extensions alone would miss it).
  - **`.ddc-keep` and excludes, by folder kind.** An exclude is `--exclude` or one from Settings.
    - **The folder itself.** A `.ddc-keep` directly in it (`~/Pictures/.ddc-keep`), or an exclude of the folder or above it, makes the whole folder `user.kept`, with no contents. Unchanged.
    - **Pictures, Movies, Music and iPhone and iPad backups**, which the scan claims whole:
      - an exclude anywhere inside still makes the whole folder `user.kept` (unchanged: the exclude stays the coarser tool);
      - a `.ddc-keep` below the folder's top locks the entry that holds it (lock "kept"), and the folder stays its rule's finding. This is the change: today such a marker turns the whole folder into `user.kept`, which would make all of Pictures unpickable.
    - **Downloads**, a region in which rules keep working:
      - a direct child folder with a `.ddc-keep` at its top, or an excluded child folder, is its own `user.kept` finding and not an entry (unchanged);
      - an excluded file is listed locked "kept": the scan's exclude claims directories only, so the lock is what protects a file;
      - a marker or an excluded folder deeper inside an entry becomes a `user.kept` finding inside it, as today, and locks the entry that holds it (lock "kept"); an excluded file anywhere inside never becomes a finding and only locks the entry.
    - **One lock for both.** "Kept" means a `.ddc-keep` or an exclude, whichever protects it.
  - **A protected folder** (safety ADR §6).
  - **Holds another volume:** an entry with a mount point inside it.
  - **In use or still arriving:**
    - an entry added (`ATTR_CMN_ADDEDTIME`) or modified within 10 minutes of the scan's start (`Env::now`), either side. A time further in the future (a camera with a wrong clock) is not "recent", so such a file does not stay locked forever. A time the filesystem does not report is left out and the other decides; with neither, the entry is locked "recent" (fails closed). For a folder this sees only changes to its direct children: an archive expanding deep inside an old folder is not seen. That limit is accepted. `Env::now` is the process start, so a rescan longer than 10 minutes can list a file changed after its start as not recent; the picked identity must still match, so this is accepted. The dry run, which has no scan, measures against its own start; the window shows locks as of its last scan until Scan now, and the clean's fresh scan decides;
    - a partial download (`*.download`, `*.crdownload`, `*.part`);
    - a backup whose `Status.plist` does not say it finished. A missing or unreadable `Status.plist` locks the backup (fails closed).
- [x] **No contents at all** for these findings, and none for the cloud folders:
  - `user.telegram` and `user.whatsapp`: their files belong to the app's database, so the app's own cache settings clean them;
  - `user.kept`: the user's own protection;
  - the cloud folders are a skip, not a finding: still not traversed, because a file there may live only in the cloud, and the Trash would remove it from every device;
  - the `trash` rule (the Trash itself).
- [x] **Downloads and other findings.**
  - Another rule's finding inside Downloads is not a Downloads entry: a downloaded `node_modules` stays its own finding in its own group, and a folder with `.ddc-keep` at its top is a `user.kept` finding (above).
  - An application bundle (`Foo.app`), which the scan claims without listing it as a finding, is an ordinary entry: its bytes fold into the entry, it is pickable, and it does not count as a finding inside. The in-use check skips it while it runs.
  - A folder entry that holds such findings is listed with its whole size (their bytes folded back in), and says how many findings it holds.
  - The Downloads group's own size is still the finding's bytes (the remainder), for the sidebar order and "selected of total", so nothing counts twice across groups.
  - A folder entry with a `.git` is listed with the note "a project (has .git)", and stays pickable: the Trash keeps it.
- [x] **A backup entry** shows the device name (`Device Name`) and the date of the last backup, from the backup's `Info.plist`, when they can be read.
  - The plists are read by a small bounds-checked reader of binary and XML property lists: top-level keys with string and date values only, never through a symlink, with capped reads, failing closed. Not the `plist` crate: the scan needs three keys, and a new crate could not be fetched offline when the done story "CLI: clean an explicit item list" tried one.

**CLI: what the clean accepts**

- [x] **A `user-data` item in `clean --items` gets its own checks.** They replace the rule-marker checks of the safety ADR's §5, which apply to directories found by rules. The executor relaxes the `none` method only for an entry of one of the five findings, never for `none` in general: the Trash, the messenger containers and iMazing backups are `none` too.
  - **Folder:** the item's parent is one of the five folders, the item's `rule` is that folder's rule, and the parent's identity equals that of the fresh scan's finding. Otherwise the item is skipped as `not-an-entry`, or as `kept` when the whole folder is kept now.
  - **Entry:** a regular file or directory, never a symlink or another kind, opened through the parent's file descriptor with the same `dev` and `ino` as listed.
  - **Still listed:** the fresh scan still holds it as an unlocked entry. The check runs against the folder's full list of entries, kept in memory, not against the report's top 100, so new large downloads cannot push a picked entry out.
  - **Unchanged from §5:** every other check still runs: inside the scan roots, no symlink above the parent, protected paths, in use, `.ddc-keep`. Only four are replaced: the rule-marker match (`verify`), the `.git` or project-root check, the `incomplete` check and the directory-only check.
  - **Skip reasons:** three new ones. A picked entry that is no longer an entry is skipped as `not-an-entry`; one that is locked now is skipped as `entry-locked`, with its lock reason (app-managed, kept, protected, other volume, recent, partial download, unfinished backup); an item inside an entry that went to the Trash is skipped as `inside-another-item` (see Nested items).
  - **Order of the checks:** the action and what the fresh scan says of the path itself (the folder, a `user.kept` entry, a kept folder) come first, then the path checks every item gets (inside the scan roots, no symlink above the parent), then the entry checks, then the lock, then the rest of §5. The executor derives the lock again with the same `lock_of`: the plan is not trusted input. So the dry run and the clean give the same reason: an entry replaced since the scan is `changed-since-scan`, not `recent`, and a marker in or at an entry, or an exclude at or in a Downloads entry, is `entry-locked` (kept), never the plain `kept` that §5's marker walk would give. A Downloads folder that became a `user.kept` finding this way is `entry-locked` too, not `not-an-entry`: that check comes first. In the four folders the scan claims whole, an exclude anywhere inside keeps the whole folder, so its entries are skipped as `kept`.
  - **The clean report.** These are new `SkipReason` values. Following the app's own rule that a new reason comes with a new schema version, `CLEAN_SCHEMA_VERSION` is raised and `docs/clean.schema.json` gains them. `CleanReport.swift`, which refuses an unknown reason, and `CleanWording` learn them in the same change.
- [x] **Refusals, each as the done story "CLI: clean an explicit item list" already does them:**
  - `action: delete` on a `user-data` item is skipped as `action-not-allowed`, and the rest of the list runs;
  - `permanent: true` on a `user-data` item refuses the whole list when it is read (exit 2, nothing runs), as for any item that is not `review`;
  - `--permanent` together with `--items` is refused by the argument parser (exit 2).
- [x] **Nested items.** The app never sends both a folder entry and a finding inside it, because the outer one covers the inner. The sheet says so: "includes 1 item selected on its own; it goes to the Trash with the folder, and its space returns when the Trash is emptied."
  - If a list holds both, the CLI runs the entry first. It skips the inner item as `inside-another-item` only when the entry went to the Trash; if the entry was skipped or failed, the inner item runs (after a cancel, neither runs).
  - Entry first is deliberate: the user picked the folder for the Trash, and Put Back must bring it back whole. A cache inside it then frees its space only when the Trash is emptied.
  - After the clean, the window shows a finding inside a trashed entry as cleaned, like the entry.
- [x] **The dry run** predicts these items and their skips like any other.
- [x] **CLI text output** escapes control and bidirectional format characters in the names it prints, as the app does: Downloads names become first-class CLI output here.
- [x] **ADRs.**
  - The safety ADR's amendment of 2026-10-20 states these rules.
  - The menu bar ADR, §6 item 1 (matching) and §7 (refusals), says how a `user-data` entry is matched and that it is the one exception.

**App**

- [x] **Sidebar: a User data section**, on a grey tint, ordered by size like the other classes ("Review window: sorted sidebar, Select all and folder groups").
  - It has one entry per listed user-data rule found. Downloads, Pictures, Movies, Music and iPhone and iPad backups have pickable entries; iMazing backups, Telegram, WhatsApp, Xcode archives, editor workspace storage and Protected by you are report only, as today.
  - "Cloud folders" is the last entry and reads "not scanned". No finding stands behind it: the app makes it from the report's skipped areas under `Library/CloudStorage` and `Library/Mobile Documents` (reason `excluded`), told apart from the background-skip areas by path.
  - If the report came from a background scan that skipped the protected areas, the section says that a Scan now lists these folders. The signal is the report's list of skipped areas, which `ScanReport.swift` learns to decode with all four skip reasons. The background scan skips these areas with reason `excluded`, the same as the cloud folders, and for iPhone and iPad backups it skips the parent `Library/Application Support/MobileSync`; the app matches by path prefix.
- [x] **A user-data group's view.**
  - It shows the listed entries, largest first, each picked one by one: no Select all, no heading checkbox, nothing picked by default.
  - A row says "last changed N days ago", and for a backup, the device name and the date of the last backup.
  - A locked entry shows a lock and its reason.
  - The unlisted rest is one line ("1,904 smaller items (2.3 GiB) are not listed").
  - Names are shown escaped: control characters and bidirectional format characters (U+202A–U+202E, U+2066–U+2069, and the marks U+200E, U+200F, U+061C) are written as escapes, so a download named to look like a `.pdf` cannot hide an `.exe`.
  - "Selected of total" for Downloads counts a picked folder entry without the findings inside it, ignored ones included, and never shows more than the total. The Downloads total is the remainder, which leaves out every nested finding, so the selected amount leaves out the same.
  - Cloud folders show a banner explaining why they are not scanned, pointing to Remove Download in Finder to free space.
  - "Protect this folder" applies to folder entries of Downloads, Pictures, Movies and Music only. A file entry has no protection action, because `.ddc-keep` lives in a directory; a backup entry has none, because a stray file inside a device backup is not known to be harmless to a restore or to the next backup (the whole Backup folder stays protectable as a finding, as today).
    - Its outcome differs by folder kind. In Pictures, Movies and Music the entry stays in its group, locked "kept", after the next scan. In Downloads it becomes a `user.kept` finding: it moves to "Protected by you", and its bytes leave the Downloads total.
    - The app holds the row locked until a scan reports it protected: a `user.kept` finding at its path, or an entry row locked "kept".
- [x] **Never elsewhere.**
  - User files are not in the Items view.
  - They are not in the popover's "Selected to clean", Biggest finds or notification amounts.
  - The scheduled path never touches them.
  - `ReviewItems.Lock.userData`'s text ("Your data · report only") is reworded for what is still report-only.
- [x] **Confirmation sheet.**
  - A block of its own, "Your own files", comes first, with its count and size, and the line "They go to the Trash; nothing of yours is deleted permanently." `Confirmation` therefore orders user data first, not last.
  - These items have no "Delete permanently" switch, and the preference never applies to them.
- [x] **Result.** User files count under "to the Trash", and the journal lists them.

## Implementation Plan

Planned by `projectstore:planner` against the codebase after "Review window: sorted sidebar, Select all and folder groups" landed. No spec covers the story; the contract is the safety ADR's amendment of 2026-10-20 and the menu bar ADR §6 item 1 and §7. CLI first, then DDCKit, then the app.

**Core and CLI** (`crates/ddc-core`, `crates/ddc-cli`)

1. **Times for one listing.** `scan/reader.rs`, `sys/bulk.rs`: `EntryMeta` gains `modified` and `added`; a `DirReader::read_with_times` (default: `read`) asks `getattrlistbulk` for `ATTR_CMN_MODTIME` and `ATTR_CMN_ADDEDTIME` only for the five folders' own listings. A `sys` helper reads one child's times through the parent's descriptor, for the dry run.
2. **Walker.** `scan/mod.rs`, `scan/walker.rs`, `scan/inodes.rs`: a `Visitor::lists_entries` hook; per-entry accumulators for each direct child of those folders, inherited by deeper tasks and across a nested claim, listed or not (the folded-back size; a `.app` included); per-entry errors, skipped and other-filesystem counts; hard links counted for an entry only when every link is inside it. Output sorted by name bytes, so two scans give the same report.
3. **What locks an entry.** New `rules/entries.rs`: the five rule ids, the app-managed extensions and fixed paths, the partial-download suffixes, the 10-minute window against `Env::now()`, the `EntryLock` enum and one `lock_of` used by the scan and the dry run; an exclude at or inside an entry locks it "kept". `rules/classify.rs`: a marker below the top of one of the four claimed folders no longer converts it to `user.kept`; `~/Pictures/.ddc-keep` and excludes still do (`claim_unless_kept` and the exclude branch are unchanged). In Downloads, a `user.kept` finding inside an entry locks it.
4. **Backup plists.** New `report/backups.rs`: a small bounds-checked reader for binary and XML plists (no new crate), opened without following symlinks; `Status.plist`'s `SnapshotState` and `Info.plist`'s device name and last backup date.
5. **`contents` and the full list.** `model.rs` (`Contents`, `FolderEntry`), new `report/contents.rs`, `report/mod.rs` (`scan_with_entries` returns the report and an in-memory `EntryIndex` of every entry with its state); `docs/scan.schema.json` regenerated, `SCHEMA_VERSION` unchanged.
6. **Plan.** `plan.rs`: `Action::TrashEntry` (its kind is `trash`), produced only by `plan_listed` for a `user-data` item, through the ordered checks above; never by `build_plan`, so `--yes` and `--class` cannot reach user data. Entries come first in the plan.
7. **Executor.** `exec/mod.rs`, `exec/guard.rs`, `sys/fsops.rs`: the three skip reasons; `TrashEntry` only with listed consent and one of the five rules; a `check_entry` sharing the §5 prefix and replacing the four checks; `Verified` keeps the checked kind, so a file entry is not mistaken for a changed directory; an item inside a trashed entry is skipped.
8. **Dry run.** `exec/preflight.rs`: the same checks and reasons, with the entry's lock computed from its own listing, marker walk and `Status.plist`; lines stay in list order.
9. **Clean report and CLI.** `exec/report.rs`, `docs/clean.schema.json` (`CLEAN_SCHEMA_VERSION` 2), `cmd/clean_items.rs`, `prompt.rs`, `render.rs` (no more "never cleaned" for what can now be picked; names escaped).

**DDCKit** (`app/Packages/DDCKit`)

10. `ScanReport.swift` decodes `contents`, `EntryLock` (strict) and the skipped areas; `CleanReport.swift` the three reasons at version 2; `CleanWording` words the reasons, the locks, the user-data group names and the entry rows.
11. `ReviewItems.swift`: an entry becomes a row whose finding is synthesized from it (the folder's rule, class `user-data`, removal `trash`), with ids after the findings'; group totals, the Items view and `locking` leave entry rows out or carry them.
12. `GroupLayout.swift` (an entries shape and a not-scanned shape, a rest line), `ReviewModel.swift` (the Downloads amount, no double count in the footer, no protection for file or backup entries, findings inside a trashed entry shown as cleaned), `Confirmation.swift` (user data first, findings inside a picked entry dropped), `Formatting.swift` (bidi escapes).

**App** (`app/DeepDriveCleaner`)

13. `ReviewView.swift`, `ReviewGroupsView.swift`, `ConfirmSheet.swift`, `PopoverView.swift`: the User data section and its rows, the cloud banner, the background-skip line, the "Your own files" block, and the old "never cleaned" and "report only" wording.

**Fixtures are "recent" when written.** Core tests run the scan with `Env::now()` an hour ahead (`Env::with_paths`) and set a "changed a minute ago" time relative to it; the CLI and real-CLI tests backdate both times through `setattrlist` (`ATTR_CMN_MODTIME` and `ATTR_CMN_ADDEDTIME` are settable): a public `sys` helper in Rust (public so the CLI's integration tests can use it), Darwin's `setattrlist` in Swift. In the CLI tests a fresh fixture is the "recent" case; in the core tests, with the clock an hour ahead, it is not, and the "recent" case sets a time a minute before the shifted start.

**Tests**: `crates/ddc-core/tests/user_data.rs` (new) and the scanner, rules, report and clean-items suites with `FakeTrash` and temporary homes; `crates/ddc-cli/tests/clean_items_cli.rs` for the refusals; `UserDataTests.swift` (new), `ContractTests.swift`, `RealCLIDryRunTests.swift` with a home of its own, and the version-2 updates to `CleanResultTests.swift` and its fixtures. Then `make check`, `make app-test`, and `projectstore:reviewer`.

## Acceptance Criteria

- [x] Rust test: the scan lists entries over a temporary home.
  - The Downloads, Pictures and Movies entries come largest first, with the cap and the floor, and the rest as a count and bytes.
  - Symlinks and dataless entries are not listed.
  - A `.photoslibrary` entry, `Music/Music`, an entry holding `.ddc-keep` (while Pictures stays `user.pictures`), an entry modified a minute ago, a `*.part` file and a backup without a finished `Status.plist` are listed locked.
  - Telegram, WhatsApp, iMazing backups, `user.kept` and the Trash list no contents, and the cloud folders are still not traversed: they appear only in `skipped`.
  - A Downloads folder holding a `node_modules` finding is listed with the folded-back size and the count of findings; a `Foo.app` in Downloads is a pickable entry with its whole size and no findings counted.
  - Markers and excludes by folder kind: `~/Pictures/.ddc-keep`, or an exclude inside Pictures, still makes all of Pictures `user.kept`; a marker inside a Pictures entry locks it "kept"; in Downloads, a child folder with `.ddc-keep` at its top, or an excluded child folder, is a `user.kept` finding and not an entry, an excluded file is listed locked "kept", and an entry with a marker deeper inside is locked "kept".
  - An entry changed a minute before the scan's start is locked "recent"; one changed an hour before it is not, nor is one dated a day ahead.
  - An entry with an unreadable part is listed "partly unreadable" and pickable; an entry where the reader reports another filesystem is locked (a fake reader).
  - A backup shows its device name and last backup date; `user.imazing-backups` lists no contents.

  Evidence: `crates/ddc-core/tests/user_data.rs::the_scan_lists_entries_largest_first_with_the_cap_and_the_floor_and_sums_the_rest`, `::the_default_limits_list_a_hundred_entries_of_at_least_ten_mebibytes`, `::locked_entries_are_listed_with_their_reason`, `::a_fresh_entry_is_recent_against_the_scans_start`, `::a_marker_at_the_folders_top_or_an_exclude_inside_still_keeps_the_whole_folder`, `::a_downloads_folder_holds_findings_with_their_size_and_an_app_is_an_ordinary_entry`, `::backups_show_their_device_and_lock_until_they_finished`, `::messengers_imazing_kept_and_the_trash_list_nothing_and_cloud_folders_stay_closed`, `::an_entry_with_an_unreadable_part_is_listed_partly_unreadable_and_stays_pickable`; `crates/ddc-core/src/report/contents.rs::what_the_walk_could_not_see_decides_the_state` (another volume locked, dataless and symlink entries not listed, with a fake reader).
- [x] Rust test: the plist reader reads `SnapshotState` from binary and XML `Status.plist` fixtures (finished, and not), and `Device Name` and `Last Backup Date` from `Info.plist`; a truncated, garbage or missing file reads as not finished, without a panic. — evidence: `crates/ddc-core/src/report/backups.rs::a_finished_backup_reads_as_finished_in_both_formats`, `::the_device_and_last_backup_come_from_the_top_level_only`, `::missing_truncated_or_garbage_files_read_as_not_finished_without_a_panic`, `::a_symlink_or_a_fifo_is_never_read`
- [x] Rust test (the core's `FakeTrash`, never the real Trash): `clean --items` moves a listed user-data entry, file and folder, to the Trash. It also gives each of the following its expected outcome:
  - a symlink entry, an entry replaced since the scan (new `ino`), an entry that is now locked (a marker put inside it: `entry-locked`, kept), and the folder itself: skipped with their reasons;
  - `--exclude` of a Downloads folder entry, and of a file inside another: `entry-locked` (kept), never the plain `kept`; an exclude inside a Pictures entry keeps all of Pictures: `kept`;
  - a path that is not an entry (a file two levels down): skipped as `not-an-entry`;
  - an entry that moved out of the top 100: still accepted;
  - `action: delete`: skipped as `action-not-allowed`;
  - `permanent: true`: the list is refused (exit 2);
  - `--permanent` with `--items`: refused (exit 2);
  - a finding inside a picked folder entry: skipped as `inside-another-item` when the entry went to the Trash, and run when the entry was skipped;
  - a symlink above the parent, and a path outside the scan roots: skipped as for any item;
  - `build_plan` over a scan with `contents` plans no entry and no user-data finding, and `execute` with `--yes` consent touches none.

  The dry run predicts the same outcomes. It has no scan, so it cannot see that a direct child of Downloads became another rule's finding: there it predicts a move to the Trash, and the clean's rescan skips it as `not-an-entry`.

  Evidence: `crates/ddc-core/tests/user_data.rs::a_picked_file_and_folder_go_to_the_trash`, `::a_symlink_a_replaced_entry_a_locked_one_and_the_folder_itself_are_skipped`, `::an_exclude_locks_a_downloads_entry_and_keeps_a_folder_claimed_whole`, `::a_path_that_is_no_entry_is_skipped_and_one_beyond_the_listed_hundred_is_not`, `::another_rules_finding_in_downloads_is_no_entry_in_the_clean`, `::delete_and_permanent_are_refused_for_a_picked_entry`, `::a_finding_inside_a_trashed_entry_is_skipped_and_runs_when_the_entry_was_skipped`, `::a_symlink_above_the_folder_or_a_path_outside_the_roots_is_skipped_as_for_any_item`, `::a_plan_never_holds_an_entry_and_yes_touches_none`, `::the_executor_derives_an_entrys_lock_again_right_before_the_trash` (each through the dry run and the clean); `crates/ddc-cli/tests/clean_items_cli.rs::items_with_permanent_or_a_malformed_list_are_refused_untouched` (`--permanent` with `--items`, exit 2).
- [x] Contract test: the JSON schemas cover the scan report's `contents`, a `user-data` item of the `--items` list, and the clean report's new skip reasons at the raised `CLEAN_SCHEMA_VERSION`; `CleanReport.swift` decodes every reason and every lock and `CleanWording` words them; every listed user-data rule of the catalog has a sidebar name — evidence: `crates/ddc-core/tests/report.rs::committed_json_schema_is_current`, `crates/ddc-core/tests/clean_items.rs::committed_clean_schema_is_current` (the report, the item list and the dry-run lines); `ContractTests.cleanContracts`, `.schemaEnums`, `.skipReasonsHaveText`, `.userDataNames`; `UserDataTests.locks`
- [x] DDCKit unit tests:
  - a user-data row is selectable only when listed and unlocked, and never selected by default;
  - its view has no Select all;
  - the item list sends `class: user-data`, `action: trash` and `permanent: false` whatever the preference;
  - picking a folder entry leaves out a finding inside it;
  - the confirmation lists these items in their own block, first;
  - a name with a bidirectional format character is shown escaped;
  - the Downloads "selected of total" leaves out findings inside a picked entry and never exceeds the total;
  - entries never reach the popover's Selected to clean, Biggest finds or notification amounts;
  - the background-skip line shows when the report skipped the protected areas;
  - the Cloud folders row comes from `skipped` by path: `~/Library/CloudStorage` (excluded) gives the row; `~/Downloads` and `~/Library/Containers` (excluded, a background scan) give the Scan-now line and no Cloud row;
  - after a clean, a finding inside a trashed entry shows as cleaned.

  Evidence: `UserDataTests.selectable`, `.layout`, `.listed`, `.confirmation`, `.escaped`, `.amounts`, `.popover`, `.skippedAreas`, `.cleanedInside`; `ReviewGroupsTests.userDataNoSelectAll`.
- [x] Real-CLI dry run over a temporary home: a backdated Downloads entry the window lists is accepted (exit 0, would move to the Trash), and a fresh one would be skipped as `entry-locked` (recent) — evidence: `RealCLIDryRunTests.picksADownload`; in the CLI's own tests, `crates/ddc-cli/tests/clean_items_cli.rs::a_picked_download_would_go_to_the_trash_unless_it_is_recent_and_names_are_escaped`
- [ ] Manual (owner, moved to the owner-checks runbook at close, 2026-10-31):
  - pick a file in Downloads, clean, find it in the Trash and put it back;
  - Photos Library shows as locked;
  - Cloud folders explains why they are not scanned.

## Final Summary

**What changed.** The user can pick their own files in five user-data folders and move them to the Trash from the Review window. Shipped in 0.2.0.
- The scan lists the entries of Downloads, Pictures, Movies, Music and iPhone and iPad backups as an optional `contents` list (largest first, at most 100 of at least 10 MiB, the rest summed), with a lock reason where one applies: managed by an app, kept, protected, another volume, recent, a partial download, an unfinished backup. Messengers, iMazing backups, `user.kept`, the Trash and the cloud folders list nothing.
- `clean --items` accepts a picked entry to the Trash only, checked against the fresh scan's full list of that folder and its identity; three new skip reasons (`not-an-entry`, `entry-locked`, `inside-another-item`) at a raised clean-report schema version. `delete`, `permanent` and `--permanent` are refused.
- A bounds-checked reader of binary and XML property lists for the backups' `Status.plist` and `Info.plist`.
- The app: a User data section in the sidebar, entries picked one by one with escaped names, a "Your own files" block first in the sheet with no Delete permanently, Cloud folders explained, nothing of this in the popover's amounts or the scheduled path.
- The safety ADR's amendment of 2026-10-20 and the menu bar ADR's §6 and §7 state the rules.

**Why.** The owner's decision of 2026-10-20: the Review window opens User data, and the user picks their own files for the Trash, rather than browsing only or treating them as ordinary `review` items.

**Tests.** The evidence above; the Rust tests run in the Gate's `gate` job on macOS and the DDCKit tests in its `app` job, both green on main at 2cdae98 (2026-10-31). Closed under the owner's rule of 2026-10-30: the manual check moved to the runbook "Owner checks: manual acceptance across stories".

**Risks and follow-ups.**
- `user.imazing-backups` stays report only until its backup layout is checked on a Mac with iMazing.
- "Recent" is judged against the start of the scan; a folder entry sees only changes to its direct children.
- A later story, "User data: move a folder's picked files to the Trash from that folder" (DDC-002), added a scoped button per folder.

## Technical Notes

- **Walker changes.** `scan/walker.rs` sums bytes per owner (`OwnerAcc`), not per direct child, so the planner should scope these changes:
  - per-entry accumulators under the five user-data owners;
  - folding nested claims back into the entry that holds them (under the Downloads region a claimed subtree gets a new owner);
  - hard-link holders per entry, as `finish()` does per owner;
  - `ATTR_CMN_MODTIME` and `ATTR_CMN_ADDEDTIME` requested from `sys/bulk.rs` only for the five folders' direct children.

  This is one pass, with no second walk.
- **Classification.** `rules/classify.rs` (the keep marker inside a claimed finding) changes for the four claimed folders only: a marker below the top locks the entry that holds it ("kept").
- **Open files.** The in-use check (`exec/guard.rs`) sees processes' executables, arguments, working directories and environment, not their open files. A `.mov` open in an editor can therefore be moved to the Trash; the Trash keeps it. Locking an entry modified in the last 10 minutes narrows this.
- **Without Full Disk Access.** `clean --items` from Terminal rescans Downloads, Pictures and the iPhone backups under `MobileSync`, so macOS may ask for access, as any scan there does. A rescan that cannot read a folder holds no entries in it, so a picked entry there is skipped as `not-an-entry`. (Note, 2026-10-24: once story "Faster clean: re-check only what the selection needs" lands, only a list holding a picked entry walks that entry's folder; a list of caches reads none of these folders.)
- **Tests and the Trash.** Tests never use the real Trash: the core tests trash through `FakeTrash` (`crates/ddc-core/tests/common/fakes.rs`), and the CLI-level tests run dry.
- **Never the real home.** The real-CLI test runs over a temporary home (`HOME` and `DDC_DATA_DIR` set by a wrapper), as "Review and clean window" does.
- **Decided at planning (2026-10-20),** the conservative choice each time:
  - markers and excludes by folder kind, as above: only a marker below the top of a claimed folder changes, and one lock, "kept", covers a marker and an exclude, so an excluded file in Downloads is protected too;
  - "recent" is measured against the scan's start, and a time the filesystem does not report leaves the other to decide (neither: locked);
  - iMazing backups stay report only until their layout is checked;
  - an application bundle in Downloads is an ordinary entry;
  - more app-managed packages and paths (Lightroom, Aperture, Capture One, Motion templates);
  - three skip reasons, not two;
  - Xcode archives and editor workspace storage get sidebar names, report only;
  - the Downloads amount leaves nested findings out and is clamped;
  - path checks run first, so the dry run and the clean agree;
  - a missing `Status.plist` fails closed, and a small plist reader, not a crate;
  - "recent" is 10 minutes either side;
  - names escape bidirectional format characters, in the app and in the CLI's text output;
  - the entry runs before a finding inside it.
- **Follow-ups.** More app-managed paths as they turn up; iMazing backups' contents once their layout is checked on a Mac with iMazing; cloud folders' Remove Download (`brctl evict`) as its own action, not the Trash.

## Dependencies

- "Review window: sorted sidebar, Select all and folder groups" (the sidebar and rows this story adds to)
- "CLI: clean an explicit item list" (done)
- ADR "Cleanup safety model: risk classes, plan and confirmation" (amended 2026-10-20)

## Attachments

- Design canvas, board 9: https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV

---

*Last updated: 2026-10-31*

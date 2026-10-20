---
type: story
id: "story-user-data-pick-your-own-files-for-the-trash"
epic: "DDC-002"
title: "User data: pick your own files for the Trash"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-20
updated: 2026-10-20
external_refs: {}
tags: [cli, app, safety, user-data]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# User data: pick your own files for the Trash

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner's decision of 2026-10-20: the Review window opens User data, and the user can pick their own files and move them to the Trash.

Until now user data never entered a plan: the safety ADR, §1, gives it the `none` method, and the executor refuses it. This story lets hand-picked entries inside six user-data folders into a plan, to the Trash only, and keeps the rest of user data out. The owner chose this over two alternatives:
- browsing only, with deletion left to Finder;
- treating user data as ordinary `review` items, permanent deletion included.

The rules are in the safety ADR's amendment of 2026-10-20, and board 9 of the design canvas draws the screen.

**Terms.** An **entry** is a direct child of one of the six folders (for the backup rules, one backup folder). The **folder** is the user-data finding it lies in.

## Decomposition

**CLI: what the scan lists**

- [ ] **`scan` lists each folder's entries.**
  - It covers each finding of `user.downloads` (the remainder), `user.pictures`, `user.movies`, `user.music`, `user.ios-backups` and `user.imazing-backups`.
  - The report lists the finding's entries as an optional `contents` list.
  - Each entry carries its path, bytes, `dev` and `ino`, kind (`file` or `folder`), modification time, and a lock reason when it has one.
  - Entries come largest first: at most 100, each at least 10 MiB, and the rest as a count and a byte total.
  - Only regular files and directories are listed: never a symlink or another kind, and never a dataless (`SF_DATALESS`) entry.
  - An entry whose own metadata cannot be read is not listed. An entry with parts it may not read is listed with "partly unreadable", its size a lower bound, and stays pickable: the Trash keeps it.
  - The field is additive. `SCHEMA_VERSION` of the scan report stays, because the Swift decoder ignores unknown keys (verified). `LISTED_SCHEMA_VERSION` of the item list stays as well, because the app and the CLI ship together.
- [ ] **Locked entries** are listed with their reason, and can never be picked.
  - **Managed by an app**, from a maintained list, named by the app:
    - library packages by extension: `.photoslibrary`, `.photolibrary`, `.musiclibrary`, `.tvlibrary`, `.imovielibrary`, `.fcpbundle`, `.lrcat`, `.lrdata`;
    - fixed paths: `Music/Music`, `Music/iTunes` (a library upgraded from older macOS keeps its media there), `Music/GarageBand`, `Music/Audio Music Apps`, `Movies/TV` and `Pictures/Photo Booth Library`.
  - **Protected by `.ddc-keep`:** an entry with a `.ddc-keep` in it or anywhere below it. For these six rules, the marker locks the entry that holds it, and the folder stays its rule's finding. Today a marker anywhere in a claimed finding turns the whole finding into `user.kept`, which would make all of Pictures unpickable. That classification changes for these six rules only.
  - **A protected folder** (safety ADR §6).
  - **Holds another volume:** an entry with a mount point inside it.
  - **In use or still arriving:**
    - an entry added (`ATTR_CMN_ADDEDTIME`) or modified in the last 10 minutes. For a folder this sees only changes to its direct children: an archive expanding deep inside an old folder is not seen. That limit is accepted;
    - a partial download (`*.download`, `*.crdownload`, `*.part`);
    - a backup whose `Status.plist` does not say it finished.
- [ ] **No contents at all** for these findings:
  - `user.telegram` and `user.whatsapp`: their files belong to the app's database, so the app's own cache settings clean them;
  - `user.kept`: the user's own protection;
  - `user.cloud`: still not traversed, because a file there may live only in the cloud, and the Trash would remove it from every device;
  - the `trash` rule (the Trash itself).
- [ ] **Downloads and other findings.**
  - Another rule's finding inside Downloads is not a Downloads entry: a downloaded `node_modules` stays its own finding in its own group, and a folder with `.ddc-keep` is a `user.kept` finding.
  - A folder entry that holds such findings is listed with its whole size (their bytes folded back in), and says how many findings it holds.
  - The Downloads group's own size is still the finding's bytes (the remainder), for the sidebar order and "selected of total", so nothing counts twice across groups.
  - A folder entry with a `.git` is listed with the note "a project (has .git)", and stays pickable: the Trash keeps it.
- [ ] **A backup entry** shows the device name (`Device Name`) and the date of the last backup, from the backup's `Info.plist`, when they can be read.

**CLI: what the clean accepts**

- [ ] **A `user-data` item in `clean --items` gets its own checks.** They replace the rule-marker checks of the safety ADR's §5, which apply to directories found by rules. The executor relaxes the `none` method only for an entry of one of the six findings, never for `none` in general: the Trash and the messenger containers are `none` too.
  - **Folder:** the item's parent is one of the six folders, and its identity equals that of the fresh scan's finding.
  - **Entry:** a regular file or directory, never a symlink or another kind, opened through the parent's file descriptor with the same `dev` and `ino` as listed.
  - **Still listed:** the fresh scan still holds it as an unlocked entry. The check runs against the folder's full list of entries, kept in memory, not against the report's top 100, so new large downloads cannot push a picked entry out.
  - **Unchanged from §5:** every other check still runs: inside the scan roots, no symlink above the parent, protected paths, in use, `.ddc-keep`. Only four are replaced: the rule-marker match (`verify`), the `.git` or project-root check, the `incomplete` check and the directory-only check.
  - **Skip reasons:** a picked entry that is no longer an entry is skipped as `not-an-entry`; one that is locked now is skipped as `entry-locked`, with its lock reason (app-managed, keep marker, protected, other volume, recent, partial download, unfinished backup).
  - **The clean report.** These are new `SkipReason` values. Following the app's own rule that a new reason comes with a new schema version, `CLEAN_SCHEMA_VERSION` is raised and `docs/clean.schema.json` gains them. `CleanReport.swift`, which refuses an unknown reason, and `CleanWording` learn them in the same change.
- [ ] **Refusals, each as the done story "CLI: clean an explicit item list" already does them:**
  - `action: delete` on a `user-data` item is skipped as `action-not-allowed`, and the rest of the list runs;
  - `permanent: true` on a `user-data` item refuses the whole list when it is read (exit 2, nothing runs), as for any item that is not `review`;
  - `--permanent` together with `--items` is refused by the argument parser (exit 2).
- [ ] **Nested items.** The app never sends both a folder entry and a finding inside it, because the outer one covers the inner. The sheet says so: "includes 1 item selected on its own; it goes to the Trash with the folder, and its space returns when the Trash is emptied."
  - If a list holds both, the CLI runs the entry first. It skips the inner item as `inside-another-item` only when the entry went to the Trash; if the entry was skipped, the inner item runs.
- [ ] **The dry run** predicts these items and their skips like any other.
- [ ] **ADRs.**
  - The safety ADR's amendment of 2026-10-20 states these rules.
  - The menu bar ADR, §6 item 1 (matching) and §7 (refusals), says how a `user-data` entry is matched and that it is the one exception.

**App**

- [ ] **Sidebar: a User data section**, on a grey tint, ordered by size like the other classes ("Review window: sorted sidebar, Select all and folder groups").
  - It has one entry per user-data rule found: Downloads, Pictures, Movies, Music, iPhone and iPad backups, iMazing backups, Telegram, WhatsApp, Protected by you.
  - "Cloud folders" is the last entry and reads "not scanned".
  - If the report came from a background scan that skipped the protected areas, the section says that a Scan now lists these folders. The signal is the report's list of skipped areas, which `ScanReport.swift` learns to decode.
- [ ] **A user-data group's view.**
  - It shows the listed entries, largest first, each picked one by one: no Select all, no heading checkbox, nothing picked by default.
  - A row says "last changed N days ago", and for a backup, the device name and the date of the last backup.
  - A locked entry shows a lock and its reason.
  - The unlisted rest is one line ("1,904 smaller items (2.3 GiB) are not listed").
  - Cloud folders show a banner explaining why they are not scanned, pointing to Remove Download in Finder to free space.
  - "Protect this folder" applies to folder entries only. A file entry has no protection action, because `.ddc-keep` lives in a directory.
- [ ] **Never elsewhere.**
  - User files are not in the Items view.
  - They are not in the popover's "Selected to clean", Biggest finds or notification amounts.
  - The scheduled path never touches them.
  - `ReviewItems.Lock.userData`'s text ("Your data · report only") is reworded for what is still report-only.
- [ ] **Confirmation sheet.**
  - A block of its own, "Your own files", comes first, with its count and size, and the line "They go to the Trash; nothing of yours is deleted permanently." `Confirmation` therefore orders user data first, not last.
  - These items have no "Delete permanently" switch, and the preference never applies to them.
- [ ] **Result.** User files count under "to the Trash", and the journal lists them.

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

- [ ] Rust test: the scan lists entries over a temporary home.
  - The Downloads, Pictures and Movies entries come largest first, with the cap and the floor, and the rest as a count and bytes.
  - Symlinks and dataless entries are not listed.
  - A `.photoslibrary` entry, `Music/Music`, an entry holding `.ddc-keep` (while Pictures stays `user.pictures`), an entry modified a minute ago, a `*.part` file and a backup without a finished `Status.plist` are listed locked.
  - Telegram, WhatsApp, `user.kept`, `user.cloud` and the Trash list no contents, and the cloud folders are still not traversed.
  - A Downloads folder holding a `node_modules` finding is listed with the folded-back size and the count of findings.
- [ ] Rust test (the core's `FakeTrash`, never the real Trash): `clean --items` moves a listed user-data entry, file and folder, to the Trash. It also gives each of the following its expected outcome:
  - a symlink entry, an entry replaced since the scan (new `ino`), an entry that is now locked, and the folder itself: skipped with their reasons;
  - a path that is not an entry: skipped as `not-an-entry`;
  - an entry that moved out of the top 100: still accepted;
  - `action: delete`: skipped as `action-not-allowed`;
  - `permanent: true`: the list is refused (exit 2);
  - `--permanent` with `--items`: refused (exit 2);
  - a finding inside a picked folder entry: skipped as `inside-another-item` when the entry went to the Trash, and run when the entry was skipped;
  - a symlink above the parent, and a path outside the scan roots: skipped as for any item.

  The dry run predicts the same outcomes.
- [ ] Contract test: the JSON schemas cover the scan report's `contents`, a `user-data` item of the `--items` list, and the clean report's new skip reasons at the raised `CLEAN_SCHEMA_VERSION`; `CleanReport.swift` decodes every reason and `CleanWording` words it
- [ ] DDCKit unit tests:
  - a user-data row is selectable only when listed and unlocked, and never selected by default;
  - its view has no Select all;
  - the item list sends `class: user-data`, `action: trash` and `permanent: false` whatever the preference;
  - picking a folder entry leaves out a finding inside it;
  - the confirmation lists these items in their own block, first.
- [ ] Real-CLI dry run over a temporary home: a Downloads entry the window lists is accepted (exit 0, would move to the Trash)
- [ ] Manual check:
  - pick a file in Downloads, clean, find it in the Trash and put it back;
  - Photos Library shows as locked;
  - Cloud folders explains why they are not scanned.

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- **Walker changes.** `scan/walker.rs` sums bytes per owner (`OwnerAcc`), not per direct child, so the planner should scope these changes:
  - per-entry accumulators under the six user-data owners;
  - folding nested claims back into the entry that holds them (under the Downloads region a claimed subtree gets a new owner);
  - hard-link holders per entry, as `finish()` does per owner;
  - `ATTR_CMN_MODTIME` requested from `sys/bulk.rs` only for the six folders' direct children.

  This is one pass, with no second walk.
- **Classification.** `rules/classify.rs` (the keep marker inside a claimed finding) changes for the six rules only: the marker locks the entry that holds it.
- **Open files.** The in-use check (`exec/guard.rs`) sees processes' executables, arguments, working directories and environment, not their open files. A `.mov` open in an editor can therefore be moved to the Trash; the Trash keeps it. Locking an entry modified in the last 10 minutes narrows this.
- **Without Full Disk Access.** `clean --items` from Terminal rescans Downloads and Pictures, so macOS may ask for access, as any scan there does.
- **Tests and the Trash.** Tests never use the real Trash: the core tests trash through `FakeTrash` (`crates/ddc-core/tests/common/fakes.rs`), and the CLI-level tests run dry.
- **Never the real home.** The real-CLI test runs over a temporary home (`HOME` and `DDC_DATA_DIR` set by a wrapper), as "Review and clean window" does.
- **Follow-ups.** More app-managed paths as they turn up; cloud folders' Remove Download (`brctl evict`) as its own action, not the Trash.

## Dependencies

- "Review window: sorted sidebar, Select all and folder groups" (the sidebar and rows this story adds to)
- "CLI: clean an explicit item list" (done)
- ADR "Cleanup safety model: risk classes, plan and confirmation" (amended 2026-10-20)

## Attachments

- Design canvas, board 9: https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3

---

*Last updated: 2026-10-20*

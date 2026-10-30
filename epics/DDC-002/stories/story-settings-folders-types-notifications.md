---
type: story
id: "story-settings-folders-types-notifications"
epic: "DDC-002"
title: "Settings: folders, types, notifications"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-30
external_refs: {design_canvas: "https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV"}
tags: [app, ui, settings]
code_refs: ["README.md", "app/DeepDriveCleaner/AppDelegate.swift", "app/DeepDriveCleaner/AutomationSettingsView.swift", "app/DeepDriveCleaner/FoldersSettingsView.swift", "app/DeepDriveCleaner/GeneralSettingsView.swift", "app/DeepDriveCleaner/PermissionsSettingsView.swift", "app/DeepDriveCleaner/ReviewView.swift", "app/DeepDriveCleaner/ScheduleSettingsView.swift", "app/DeepDriveCleaner/SettingsWindowController.swift", "app/DeepDriveCleaner/TypesSettingsView.swift", "app/Packages/DDCKit/Sources/DDCKit/AppPreferences.swift", "app/Packages/DDCKit/Sources/DDCKit/FolderList.swift", "app/Packages/DDCKit/Sources/DDCKit/NotReadable.swift", "app/Packages/DDCKit/Sources/DDCKit/PopoverModel.swift", "app/Packages/DDCKit/Sources/DDCKit/ReviewItems.swift", "app/Packages/DDCKit/Sources/DDCKit/ScanReport.swift", "app/Packages/DDCKit/Sources/DDCKit/Schedule.swift", "app/Packages/DDCKit/Sources/DDCKit/SettingsFile.swift", "app/Packages/DDCKit/Sources/DDCKit/SettingsModel.swift", "app/Packages/DDCKit/Sources/DDCKit/SettingsStore.swift", "app/Packages/DDCKit/Sources/DDCKit/SharedSettings.swift", "app/Packages/DDCKit/Sources/DDCKit/TypesRows.swift", "app/Packages/DDCKit/Tests/DDCKitTests/SettingsTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/RealCLISettingsTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ContractTests.swift", "crates/ddc-cli/src/cli.rs", "crates/ddc-cli/src/cmd/clean_items.rs", "crates/ddc-cli/src/render.rs", "crates/ddc-cli/tests/clean_items_cli.rs", "crates/ddc-cli/tests/rules_cli.rs", "crates/ddc-cli/tests/settings_cli.rs", "crates/ddc-core/src/model.rs", "crates/ddc-core/src/report/mod.rs", "crates/ddc-core/src/rules/listing.rs", "crates/ddc-core/src/rules/visitor.rs", "crates/ddc-core/src/settings.rs", "crates/ddc-core/tests/report.rs", "crates/ddc-core/tests/settings.rs", "docs/scan.schema.json", "docs/settings.schema.json"]
specs: []
started_at: "2026-10-28T02:56:30.432Z"
closed_at: "2026-10-30T06:45:43.051Z"
plan_updated_at: "2026-10-28T02:56:30.432Z"
---

# Settings: folders, types, notifications

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

This story was first drawn as artboard 7 of the design canvas. Artboard 10 ("Settings, revised") replaced it on 2026-10-21; see the revision below.

What the CLI also reads is stored in `settings.json` (story "CLI: shared settings, types and the notify decision"), so a scan from the terminal and a scan from the app use the same folders and types. Preferences only the app uses stay in its `UserDefaults`.

### Revision after the owner's review (2026-10-21)

Artboard 10 of the design canvas ("Settings, revised") replaces artboard 7. Its tabs are General, Folders, Schedule, Automation, Types and Permissions. The old tabs map to the new ones:
- General's Scan row moves to Schedule.
- Notifications (threshold, low disk, working hours) moves to Automation, under "Notify me when".
- The badge and the size next to the icon stay in General.

The plan gate on 2026-10-28 re-cut the Decomposition and the ACs for artboard 10.

The window is an `NSTabViewController` with `tabStyle = .toolbar`, and the window title is the pane's name. A SwiftUI `TabView` in an `NSWindow` would draw segments, not toolbar tabs.

The owner decided four things.

**1. Automatic cleaning, strict variant.** (Superseded on 2026-10-28, see D3: Automation offers two choices.) "After a scheduled scan" offers three choices:
- Notify me (the default);
- Clean download caches automatically, then notify me;
- Only show it in the menu bar.

The automatic choice has these limits:
- **What it cleans.** Only `safe` groups marked Auto in Types. Auto is on by default only for caches a package manager downloads again: npm, pnpm, yarn, bun, pip, Poetry, Homebrew and CocoaPods.
- **Groups have to split first.** Today `rules --json` puts all of these in one group with Cargo, uv, Go, Gradle (`crates/ddc-core/src/rules/listing.rs`, `GROUPS`), and Types has one switch per group. So `GROUPS` splits that group into "Package downloads" and "Toolchain caches" (Cargo, uv, Go modules and build cache, Gradle), as artboard 10 draws them. Auto is then stored per group in `settings.json`.
- **What stays off by default.** These are `safe` too:
  - Toolchain caches, which an offline build needs;
  - IDE caches, because of re-indexing;
  - Xcode builds and project build caches, which are rebuilt by compute;
  - browser and app caches: they carry `owners`, so a running browser makes them skip as in use most of the day.
- **When it runs.** Only when it frees at least N GiB, counted as private bytes. Anything in use is skipped, as in 0.2.1.

Before any code it needs amendments:
- the safety ADR: §8 ("the background path contains no call to the cleanup executor") and its Alternative A;
- the menu bar ADR: §7 (the sheet's Clean is the only caller of a clean, which `ContractTests.cleanHasOneOwner` enforces) and §5 (a scheduled instance lives about a minute).

It also needs:
- how a promoted scheduled instance shows the running clean with Stop;
- the "cleaned" notification's category and what `notify ack` records;
- the rescan after the clean.

It has its own story: "Automatic cleaning of download caches after a scheduled scan" (DDC-002). This story only draws the Automation tab and stores the choice.

**2. Exclude with two modes.**
- "Count, never clean" is today's `excludes` (`user.kept`, still read and sized).
- "Skip entirely" is new: the folder is not read. It needs:
  - an additive optional `skips` in `settings.json`;
  - a `--skip` flag;
  - the report's skipped reason, so Review says "not scanned";
  - a line in safety ADR §2 item 1.

  The `--exclude` confirmation of menu bar ADR §6 item 6 does not apply to a skip.

  This story owns that CLI change. `ContractTests.settingsKeys` checks the new key against `Settings`.

**3. Protected folders, simplified.** There is no per-folder Allow…; a scheduled scan without Full Disk Access skips every prompting area anyway (`crates/ddc-cli/src/cmd/background.rs`).
- **Without Full Disk Access,** Folders shows one banner with Open Privacy Settings. Below it is a "Not readable" list: the folders the last foreground scan reported in `inaccessible`, each with Open Privacy Settings.
- **With Full Disk Access,** only a confirmation line.
- **iCloud Drive** is never walked, so it is never listed.

**4. Schedule tab.** It holds:
- Scan automatically;
- Daily or Weekly, with the day and a whole hour;
- Pause, the same `ScansPaused` as the popover;
- Next scan, from the due function of "Background scan and notifications in the app": "Today" or "Tomorrow" at the slot, and "Within the hour" after a missed slot or a Resume.


## Decomposition

Re-cut at the plan gate (2026-10-28) for artboard 10.

- [x] CLI "Skip entirely":
  - an additive `skips` list in `settings.json`;
  - `--skip PATH` on `scan` and `clean`, which only adds to the list;
  - `not_scanned` in the report and one line in the terminal;
  - the README and both schemas.

  The ADR amendments are written first.
- [x] Core: re-cut `GROUPS` to the Types rows of artboard 10 plus "Maven and test browsers" (D2): 14 switchable rows and two fixed, 16 groups.
- [x] DDCKit:
  - `SettingsFile` and `SettingsStore`, a read-modify-write that keeps every unedited key;
  - `FolderList`, `TypesRows` and `Schedule`;
  - the new `AppPreferences` keys;
  - decoding of `inaccessible` and `not_scanned`;
  - the "Not readable" list.
- [x] App:
  - an `NSTabViewController` window (`.toolbar`) with the six panes General, Folders, Schedule, Automation, Types and Permissions;
  - the seams that "Background scan and notifications in the app" fills: next scan, scheduler control, the scheduled-scan row, the migration row;
  - the "not scanned" line in Review.

## Implementation Plan

Planned on 2026-10-28 with `projectstore:planner`. No spec covers this story.

**Owner decisions taken at the gate.** The owner delegated these on 2026-10-25 and 2026-10-28; each is recorded here so it can be overturned.
- **D1.** "Scan automatically" is on by default, daily at 10:00, as artboard 10 draws it. "Background scan and notifications in the app" relies on "the first tick is due".
- **D2.** `GROUPS` follows the 13 Types rows of artboard 10, plus one more row, in one contract change. The rows are: Python environments; node_modules; Build outputs; Build caches in projects; Package downloads; Toolchain caches; Maven and test browsers; Browser and app caches; IDE caches; Xcode builds; Xcode device support; Old toolchains; ML models; VM and container images. Interrupted cleanups and Your data stay as they are, which makes 16 groups.
  - **Changed groups:**
    - **Build outputs:** `rust.target`, `maven.target`, `gradle.build`, `cmake.build`, `cocoapods.pods`.
    - **Build caches in projects:** `js.build-cache`, `python.pycache`, `python.tool-cache`, `gradle.project-cache`.
    - **Package downloads:** npm, pnpm, yarn, bun, pip, Poetry, Homebrew, CocoaPods.
    - **Toolchain caches:** Cargo, uv, Go modules, the Go build cache and Gradle, exactly the owner's 2026-10-21 list. All of them are `safe`.
    - **Maven and test browsers** (the extra row): `repo.maven`, `browsers.playwright`, `browsers.puppeteer` and `browsers.cypress`, all `review`. They were left out of the owner's list, and putting them under a Safe row would misstate their class.
    - **Xcode builds:** DerivedData and simulator caches.
    - **Xcode device support:** `xcode.device-support`.
    - **Browser and app caches:** the `app.*` rules and `generic.cachedir-tag`.
  - Labels are data, so `RULES_SCHEMA_VERSION` stays 1.
  - **The class chip.** A Types row shows one chip for each class among its switchable rules, in class order. Some groups were already mixed before this change:
    - node_modules, with an orphan `review`, reads "Rebuildable · Review";
    - IDE caches, with JetBrains old data `review`, reads "Safe · Review";
    - Browser and app caches, with `generic.cachedir-tag` `review`, reads "Safe · Review";
    - Python environments (`python.tox` `safe`) and VM and container images (`cache.colima` `safe`) show both classes too.

    The chip states the classes as they are, and no row claims to be safer than its least safe rule.
- **D3.** The per-group Auto flag is not stored here. The Automation pane draws "Clean download caches automatically" but disables it ("arrives in a later version"), so `auto` is never stored. The flag and its storage belong to "Automatic cleaning of download caches after a scheduled scan", which needs its ADR amendments first.
  - **Superseded on 2026-10-28.** The owner parked "Automatic cleaning of download caches after a scheduled scan", so the disabled choice is removed, and Automation offers two choices: Notify me and Only show it in the menu bar. `auto` is still never stored, and a stored `"auto"` still reads as Notify me (`SettingsTests.automation`). Artboard 10 of the design canvas still draws three choices; the canvas is not edited.
- **D4.** "Not scanned" is a new optional report field, `not_scanned`, not a new `SkipReason`. It lists the user's skips that the walk actually stopped at, in canonical form. The visitor records them as it records keep markers, so the list is never truncated.
  - A skip inside a claimed finding is not listed. The walk still reads that finding, and the finding is kept whole instead.
  - The prompting areas a background scan leaves out are not listed either: they are not the user's skips.
  - A new variant would bump the scan `schema_version`, and a 0.2.x report would read as unreadable right after an upgrade.
  - `skipped` is truncated at `max_recorded`, so a skip could fall out of it.
  - `skips` keeps the settings `schema_version` at 1. `deny_unknown_fields` already makes a 0.2.x CLI refuse the key and name the file, and the app omits an empty list.

**Route.**
1. **Settings and flags.**
   - `ddc-core/src/settings.rs`:
     - add `skips` (absolute paths, `default`, skipped when empty);
     - `ScanFlags.skips` is a union: file skips plus flags;
     - a missing skip is skipped with a note;
     - the skips join `options.excludes` after the `--exclude` branch, so an exclude flag can never drop them and `dropped()` never names them;
     - they also go into a new `ScanOptions.user_skips`;
     - a skip that is not a directory, or cannot be canonicalised, is refused here, beside the absolute and missing checks, so every command agrees, the dry run included. The error names `--skip` or `skips`.
2. **Scan.**
   - `report/mod.rs` uses the canonical skips from `resolve`.
   - The visitor receives the user's skips apart from the silent prompting areas and records the ones `before_read` stops at, in a `Mutex<Vec<PathBuf>>` like `markers`.
   - `model.rs` gains `ScanReport.not_scanned`, built from that record.
   - `cmd/background.rs` keeps the user's skips in their own field (`ScanOptions.user_skips`), because its struct update overwrites `skip` with the prompting areas, and only user skips are recorded.
   - `Settings::dropped()` treats a folder that a skip covers as still protected.
3. **Clean.**
   - `clean --items`, both the real clean and the dry run, refuses an item at, under or above a resolved skip as `kept` before `plan_listed`, as `preflight` does for excludes.
   - Without this the pruned re-check never sees the skipped folder, and the item would read `changed-since-scan`.
4. **CLI.**
   - `cli.rs`: a repeatable `--skip` on `ScanArgs` and `CleanArgs`, allowed with `--items`. Like `--exclude`, a missing path is an error.
   - Thread it through `cmd/{scan,clean,clean_items,background}.rs`.
   - `render.rs`: one "Not scanned:" line.
   - README: rows in the scan, clean and Settings tables.
   - Regenerate `docs/settings.schema.json` and `docs/scan.schema.json` with `DDC_UPDATE_SCHEMA=1`.
5. **Groups.**
   - `rules/listing.rs` per D2.
   - Regenerate `Fixtures/rules.json` with `rules --json`, using a temp HOME and data dir.
   - Update the labels in `GroupCleanTests` and `ReportSummaryTests`.
6. **DDCKit settings.**
   - `SettingsFile`: a Codable mirror of every Rust key, `scheduler` and `skips` included. It writes snake_case, sorted, with empty arrays and nils omitted, through `.atomic`.
   - `SharedSettings.read` decodes through `SettingsFile` and keeps its strict checks. Skips must be absolute paths, with a `strictSettings` case.
   - `SettingsStore` and the CLI's `clear_scheduler` both read-modify-write the file without a lock. Both writes are atomic renames, and the CLI writes only when it clears a stale record, so the cost is at worst one lost `scheduler` clear, which the next launch heals.
   - `SettingsStore` is the app's one writer:
     - `update` reads from disk every time;
     - it refuses to write over a malformed or newer file;
     - after a write it calls `PopoverModel.refresh()`, so ignored findings and the icon change at once.
   - `ContractTests.settingsKeys` gains `skips`.
7. **DDCKit panes' logic.**
   - `FolderList`:
     - the first added root writes `[home, new]`;
     - nested, duplicate and symlinked roots are refused by resolved path;
     - a skip or exclude at or above a root is refused;
     - the last root cannot be removed;
     - an exclude's mode moves its path between `excludes` and `skips`.
   - `TypesRows`: one row per group of switchable rules, with the class chips of D2. Off means all of the group's switchable ids are in `disabled_rules`. The help text lives in the app, keyed by label, and a contract test checks that every label has one.
   - `Schedule`: the hour is an `Int` from 0 to 23. The weekday applies only to Weekly. The status wording is "Off", "Paused", or the next-scan seam's date.
     - While the seam returns `nil` and scheduling is on, the line is hidden.
     - 0.3.0 ships only after "Background scan and notifications in the app" fills the seam.
   - `AppPreferences`:
     - setters for the size next to the icon, the badge and delete-permanently;
     - new keys `ScheduleEnabled`, `ScheduleRepeat`, `ScheduleWeekday`, `ScheduleHour`, `AfterScheduledScan` and `NotifyWorkingHoursOnly`.
   - Pause goes through `PopoverModel.setPaused`.
8. **Report decoding.**
   - `ScanReport` decodes `inaccessible` and `not_scanned`.
   - "Not readable" is `inaccessible` collapsed to its prompting area, one row per area. A Swift mirror of `PROMPTING_AREAS` has a drift test against `background.rs`.
   - The list is cached in `UserDefaults` (`NotReadableFolders`). It is replaced only by the app's own Scan Now result, never by a report loaded from disk, because a background scan without Full Disk Access skips those areas and its report cannot say so.
   - The Review window shows one "Not scanned" line listing `not_scanned`.
9. **App.**
   - `SettingsWindowController` is an `NSTabViewController` (`.toolbar`) with one `NSHostingController` per pane. The window title follows the pane, and `show(pane:)` opens a given pane.
   - The panes are `{General,Folders,Schedule,Automation,Types,Permissions}SettingsView.swift`.
   - General reuses `LoginItem.swift` for Open at login (it already handles `.requiresApproval`). It also has the "Command-line tool" row: the first of `/opt/homebrew/bin/ddc` and `/usr/local/bin/ddc` that resolves into this bundle, otherwise the bundled helper's path. No process is spawned.
   - Folders' help text says that a skip inside a cache or a project artifact keeps that whole finding.
   - `AppDelegate` passes the model, the preferences, the store, the privacy opener and the seams.
   - Settings opens from the existing gear, the right-click menu and ⌘,.
   - Full Disk Access comes from `PopoverModel.fullDiskAccess`, the popover's single probe.
   - Notification permission is read only: Settings never asks for it.
10. **Checks.** `make check`, `make app-test`, an unsigned build, then `projectstore:reviewer`.

**Seams left for "Background scan and notifications in the app":**
- `nextScan` is the closure the popover footer already uses, and it returns `nil` until that story lands.
- `SchedulerControl` is a no-op: the schedule choice is stored, and registering or unregistering is that story's job.
- The "Scheduled scan" and "Old background scan found · Replace It" rows are shown only when their seam returns a value.

Release 0.3.0 is not tagged before that story lands.

**Pitfalls:**
- An empty `roots` means home.
- The visitor compares skips by equality, so a skip must be canonical (`/tmp` versus `/private/tmp`).
- `--skip` must only add to the skips.
- Never write app-only keys, `null` arrays or an empty `skips`.
- Never overwrite a file the app cannot read.
- Button and window strings are title case (`titleCase`).
- Settings sources never name `.clean` (`cleanHasOneOwner`).
- `min_age_days` applies from the next scan, not to the loaded report, and the help text says so.
- `after_clean` carries the old `not_scanned` until the next scan.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [x] Core test (`tests/settings.rs`) — evidence: `crates/ddc-core/tests/settings.rs` `skips_add_up_protect_and_must_be_folders`, `a_malformed_file_is_an_error_naming_it`, `committed_settings_schema_is_current` (commit 6a16187):
  - `skips` must be absolute;
  - a missing skip is skipped with a note;
  - `--skip` adds to the file's skips;
  - an `--exclude` flag leaves the skips in force and never names them as dropped;
  - a skip that is a file is refused by `resolve`, and the error names the skip;
  - unknown keys are still refused;
  - the settings `schema_version` stays 1;
  - the committed settings schema is current.
- [x] Core test (`tests/report.rs`) — evidence: `crates/ddc-core/tests/report.rs` `a_skipped_folder_is_never_opened_and_is_listed_as_not_scanned`, `committed_json_schema_is_current`:
  - a skipped folder is never opened: a mode-000 directory at or under it gives no `inaccessible` entry, and nothing inside it becomes a finding;
  - `not_scanned` lists the skipped folder in canonical form, also when it was given through a symlink;
  - a skip inside a claimed finding makes that finding `user.kept` and unselected, and is absent from `not_scanned`;
  - a report without `not_scanned` still decodes;
  - the scan `schema_version` stays 1;
  - the committed scan schema is current.
- [x] CLI test — evidence: `crates/ddc-cli/tests/settings_cli.rs` `a_skip_is_not_scanned_in_every_scan` (also the dropped-exclude case), `crates/ddc-cli/tests/clean_items_cli.rs` `an_item_at_under_or_above_a_skip_is_kept_in_the_dry_run_and_the_clean`:
  - with `skips` in `settings.json`, `scan --json` lists the folder in `not_scanned` and the terminal prints the "Not scanned" line;
  - `clean --items --dry-run` of an item at, under or above a skip predicts `kept`;
  - the real `clean --items` of the same items skips them as `kept`, not `changed-since-scan`, and removes nothing;
  - an `--exclude` flag that replaces a settings exclude covered by a skip does not name that exclude as dropped;
  - `scan --background --notify json` applies the skips together with the prompting areas.
- [x] CLI test: `rules --json` gives the 16 groups — evidence: `crates/ddc-cli/tests/rules_cli.rs` `rules_json_groups_follow_the_settings_types`, `rules_json_lists_every_rule_with_its_group`; of D2 in order, with these members:
  - "Package downloads" is exactly npm, pnpm, yarn, bun, pip, Poetry, Homebrew and CocoaPods;
  - "Toolchain caches" is exactly Cargo, uv, Go modules, Go build cache and Gradle, all `safe`;
  - "Maven and test browsers" is exactly the Maven repository and the three test-browser rules;
  - every rule still has a group.
- [x] DDCKit test: a Types row's chips — evidence: `SettingsTests.swift` `typesChips`; are the distinct classes of its switchable rules, in class order. "Toolchain caches" reads Safe, and "node_modules" reads Rebuildable and Review.
- [x] DDCKit test — evidence: `SettingsTests.swift` `keepsUneditedKeys`, `omitsEmpty`, `neverOverwritesUnreadable`:
  - Settings writes `settings.json` atomically and keeps every key it did not edit, the `scheduler` record included;
  - it omits empty lists;
  - a malformed or newer file is never overwritten, and the pane says why;
  - the written file passes `SharedSettings.read`.
- [x] DDCKit test — evidence: `SettingsTests.swift` `typesSwitch`, `typesApplyAtOnce`:
  - switching a group off writes exactly its switchable ids to `disabled_rules`, and switching it back on removes them;
  - after the store's change hook, the popover summary marks those findings ignored with no rescan;
  - the same holds for "Hide items smaller than".
- [x] DDCKit test — evidence: `SettingsTests.swift` `addRoot`, `refusals`, `lastRoot`, `exclusionMode`:
  - adding `/Volumes/Work` to the default writes `[home, /Volumes/Work]`;
  - each of these is refused with a message: a folder inside a root, a root around an existing one, the same folder given twice or through a symlink, and a skip or exclude at or above a root;
  - the last root cannot be removed;
  - changing an exclude row's mode moves its path between `excludes` and `skips`.
- [x] Test with a fake CLI (evidence: `SettingsTests.swift` `scanNowTakesNoFlags`): after roots, excludes and skips are edited, Scan Now runs exactly `scan --json --progress json`, with no paths, `--exclude` or `--skip`.
- [x] Test with the real CLI (evidence: `RealCLISettingsTests.swift` `skipIsNotScanned`, ran against `target/debug`) (temp HOME and data dir; skipped without `target/debug`): a skip written by the Settings model appears in `not_scanned`, and a `node_modules` inside it is not found.
- [x] DDCKit test (`TempDefaults`) — evidence: `SettingsTests.swift` `preferences`:
  - these app preferences persist in `UserDefaults` and never appear in `settings.json`: the size next to the icon, the badge, working hours, the after-scan choice, the schedule and delete-permanently;
  - the icon switches change the icon state at once;
  - the Pause switch writes `ScansPaused`, the same key as the popover's.
- [x] Unit test (Schedule) — evidence: `SettingsTests.swift` `scheduleDefaults`, `nextScan`, `schedulerSeam`:
  - only whole hours can be chosen;
  - "Scan automatically" defaults to on, daily at 10:00 (D1);
  - the Next scan line is the seam the popover footer uses;
  - with Scan automatically off it reads "Off…";
  - while paused it reads "Paused…".
- [x] Unit test (Automation) — evidence: `SettingsTests.swift` `automation`:
  - the GiB threshold and the low-disk switch write `notify_threshold` and `notify_low_disk`;
  - with scheduled scans off, the pane says so and is disabled;
  - the automatic choice cannot be selected, and `auto` is never stored.
- [x] Unit test (Full Disk Access) — evidence: `SettingsTests.swift` `fullDiskAccess`, `notReadableAreas`, `notReadableOwnScanOnly`; `ContractTests.swift` `promptingAreas`:
  - the Folders banner and the Permissions row follow the popover's probe on a temp path;
  - "Not readable" is the report's `inaccessible` collapsed to one row per prompting area;
  - the list changes only on the app's own Scan Now result, and loading a report from disk leaves it unchanged;
  - a drift test checks the Swift list against `PROMPTING_AREAS`.
- [x] DDCKit test (evidence: `SettingsTests.swift` `notScannedLine`): a report with `not_scanned` gives the Review window's "Not scanned" line, and a report without it gives none.
- [x] Contract tests — evidence: `ContractTests.swift` `settingsKeys`, `strictSettings` (skips, null lists), `typesHelp`, `cleanHasOneOwner`, `titleCase`:
  - `settingsKeys` includes `skips`;
  - `cleanHasOneOwner` and `titleCase` stay green;
  - every group label has help text.
- [x] `make check` and `make app-test` pass. — evidence: both exit 0 on 2026-10-28 before commit 6a16187; 166 Swift tests in 21 suites; unsigned xcodebuild with no warnings; `projectstore:reviewer` found every automated AC met.
- Manual checks (3) moved to [Owner checks: manual acceptance across stories](../../../ops/owner-checks-manual-acceptance-across-stories.md) on 2026-10-30, by the owner's decision.

## Final Summary

**What changed** (code commit 6a16187, released in 0.3.0 with 9d0418c):
- A Settings window with toolbar tabs: General, Folders, Schedule, Automation, Types and Permissions.
- Core and CLI: `skips` in `settings.json`. A skipped folder is never opened, is listed in `not_scanned`, and its items are kept by the clean. `rules --json` gives the 16 groups.
- The app writes `settings.json` atomically, keeps the keys it did not edit, and never overwrites a malformed or newer file. App-only preferences stay in `UserDefaults`.
- Types and "Hide items smaller than" apply to the popover at once, with no rescan.

**Tests run:** `crates/ddc-core/tests/settings.rs` and `report.rs`; `crates/ddc-cli/tests/settings_cli.rs`, `rules_cli.rs` and `clean_items_cli.rs`; DDCKit `SettingsTests.swift`, `RealCLISettingsTests.swift` and `ContractTests.swift`. `make check` and `make app-test` passed on 2026-10-28 (166 Swift tests in 21 suites), and `projectstore:reviewer` found every automated AC met.

**Remaining risk:** three manual checks moved to the owner list: how the window opens, tabs and resizes in light and dark; the folder picker with an external volume and the links to Privacy & Security; Open at login against System Settings.

## Technical Notes

- "Menu bar popover" stores the two icon preferences in `UserDefaults` as `ShowSizeNextToIcon` (off by default) and `ShowMenuBarBadge` (on by default); the General tab's switches write those keys.
- The groups come from the CLI (`rules --json`), so a new rule appears in the right group without an app release.
- `disabled_rules` and `min_size` hide findings from selection and totals; they do not stop the scanner from walking those folders.
- Handed to "Background scan and notifications in the app", which consumes these keys at its plan gate:
  - `AfterScheduledScan` (`notify` or `menuBarOnly`): `menuBarOnly` posts nothing and only sets the badge;
  - `NotifyWorkingHoursOnly`;
  - the `Schedule*` keys.
- From "Background scan and notifications in the app": `nextScan`, `SchedulerControl`, the scheduled-scan row and the migration row are seams here (Implementation Plan); the new `UserDefaults` schedule keys are that story's input.
- The canvas's help under Open at login says it keeps "the daily scan running". With the scheduler agent (ADR §5) scans run without the login item; the help text says the icon only.

## Dependencies

- [[epics/DDC-002/stories/story-app-skeleton-and-build-pipeline|App skeleton and build pipeline]]
- [[epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision|CLI: shared settings, types and the notify decision]]
- [[epics/DDC-002/stories/story-background-scan-and-notifications-in-the-app|Background scan and notifications in the app]]

## Attachments

- Design canvas: https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV

---

*Last updated: 2026-10-30*

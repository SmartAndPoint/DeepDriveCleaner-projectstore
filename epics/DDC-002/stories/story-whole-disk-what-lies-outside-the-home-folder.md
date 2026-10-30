---
type: story
id: "story-whole-disk-what-lies-outside-the-home-folder"
epic: "DDC-002"
title: "Whole disk: what lies outside the home folder"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-30
external_refs: {}
tags: [cli, whole-disk, system-data]
code_refs: ["crates/ddc-core/src/disk/", "crates/ddc-cli/src/cmd/disk.rs", "crates/ddc-cli/src/cli.rs", "crates/ddc-cli/src/cmd/mod.rs", "crates/ddc-cli/src/progress.rs", "crates/ddc-cli/src/render.rs", "crates/ddc-core/src/exec/mod.rs", "crates/ddc-core/src/exec/system.rs", "crates/ddc-core/src/lib.rs", "crates/ddc-core/src/progress.rs", "crates/ddc-core/src/report/volumes.rs", "crates/ddc-core/src/rules/mod.rs", "crates/ddc-core/src/rules/visitor.rs", "crates/ddc-core/src/store.rs", "crates/ddc-core/src/sys/statfs.rs", "crates/ddc-core/tests/disk.rs", "crates/ddc-core/tests/data/apfs-list.plist", "crates/ddc-core/tests/common/fakes.rs", "crates/ddc-cli/tests/disk_cli.rs", "crates/ddc-cli/tests/structure.rs", "docs/disk.schema.json", "docs/progress.schema.json", "app/Packages/DDCKit/Sources/DDCKit/ProgressEvent.swift", "app/Packages/DDCKit/Sources/DDCKit/CleanProgress.swift", "app/Packages/DDCKit/Sources/DDCKit/CleanTimings.swift", "README.md"]
specs: []
started_at: "2026-10-28T09:38:52.089Z"
closed_at: "2026-10-30T06:45:44.720Z"
plan_updated_at: "2026-10-28T09:38:52.089Z"
---

# Whole disk: what lies outside the home folder

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner asked for this on 2026-10-28, ahead of "Duplicates and APFS private size" (DDC-001).

Disk Utility shows how much of the disk is used. DeepDriveCleaner explains only the home folder. On the reference machine that is 499 GB of the 794.6 GB the startup container uses. The research note [[research/what-fills-system-data-on-a-developers-mac|What fills System Data on a developer's Mac]] names the rest:
- applications;
- Homebrew and conda prefixes;
- macOS assets, simulator runtimes among them;
- `/Library`;
- temporary folders and system databases in `/private`;
- `/usr/local`;
- the other volumes: swap, Preboot, the sealed system and Recovery;
- what only root can read.

This story is the read-only report: **`ddc disk --json`**. It lists the startup container as **areas** with their sizes, what each area is, and what frees it. The research's option B applies:
- **Nothing outside the home folder is a finding.** The walk outside it is size-only, with no rule visitor. Nothing is selected and nothing is removed.
- The home folder's rules are not applied outside it. They select `__pycache__` inside Homebrew as `safe` and call the global npm folder orphaned.

Two later stories in this epic build on this one, and neither is part of this story:
- "Whole disk in the app" shows the report;
- "Whole disk: free space with the owning tool" runs `simctl runtime delete` and `brew cleanup` after a confirmation, and amends both ADRs.

Also out of scope:
- **A persistent size index kept current from FSEvents.** It is a later story with its own ADR.
- **Abandoned temporary folders of coding agents.** They are evidence for the proposed agent-footprint epic, DDC-003, which is not yet in the vault. Here they are only measured.
- **Duplicates.**

## Decomposition

Rough until the plan gate.

- [x] **The startup container and its volumes.**
  - The source is `/usr/sbin/diskutil apfs list -plist`. System tools are run by absolute path (`/usr/sbin/diskutil`, `/usr/bin/tmutil`, `/usr/bin/xcode-select`), because `diskutil` lies in `/usr/sbin`, which `SystemCommands::find` does not search, and because a `PATH` entry must never stand in for a system tool. The plist is filtered to the container whose volume is mounted at `/System/Volumes/Data`. The plist also lists the simulator runtimes' cryptex containers and mounted disk images, which are not part of it.
  - For each volume it gives the role, the bytes consumed and whether it is mounted, plus the container's "in use" and capacity.
  - If `diskutil` fails, the fallback is `statfs` per mounted volume. Used is `f_blocks − f_bfree`; the port gains `f_bfree`. Recovery is then reported as not measured.
- [x] **The home folder row.**
  - **When `last-scan.json` can be used**, its bytes are taken as they are. The report must have the current schema, its `scan_roots` must be exactly the home folder, it must lie on the Data volume, and it must be at most 24 hours old.
  - **Otherwise `ddc disk` sizes the home folder itself.** The walk is size-only, but it keeps what the scan never opens: the cloud folders (today the catalog's `NoTraverse` matcher) and the user's `skips` (today in the `RuleVisitor`). Neither is in `WalkOptions`. The plan gate chose a size-only visitor that keeps both, through a check shared with the `RuleVisitor`, so the promise lives in one place; moving both into `WalkOptions` would change the walker that scan, recheck and the executor share. A plain `NullVisitor` walk of the home folder is not allowed, because it would open the folders the safety ADR promises are never opened.
  - **The row's date** is the report's `generated_at`. After a clean that edited `last-scan.json` in place, the bytes can be newer than that date; the row says "from the scan at …" anyway, which errs on the old side.
  - **The row carries its source and date.** The source is `last-scan` or `walk`.
  - **What the home walk does not read becomes named rows inside "not counted", never silent:**
    - the cloud folders, `~/Library/CloudStorage` and `~/Library/Mobile Documents`;
    - the user's `skips`;
    - unreadable folders, with their count.
- [x] **Areas outside the home folder.** A fixed table in code gives each area its paths, title, explanation and optional action id. All paths go through `/System/Volumes/Data/...`, because the firmlinks would otherwise count `/Applications`, `/Library`, `/opt`, `/private`, `/usr/local` and `/System/Library/AssetsV2` twice; the one exception is the home folder's own walk, which uses `env.home()` (see the plan).
  - **The size-only walk** uses the walker `ddc du` uses, with a visitor that only measures: it never claims a folder, never classifies and creates no findings. `ddc du`'s `NullVisitor` itself cannot be used: the walker drops roots nested in other roots (`/Library/Updates` in `/Library`), and the budget, the home folder's place in `/Users` and the skips each need a visitor. Each inode is counted once, unreadable folders and other file systems are recorded, and nothing is classified.
  - **Temporary folders** (`/private/var/folders`, `/private/tmp`) are walked under a budget of entries, and no single directory of more than 10,000 entries is read. Past the budget the row is marked partial ("at least"), with the entries read. On the reference machine one `$TMPDIR` holds about 10 million directories, which made a whole-disk walk take 10.5 minutes.
  - **Simulator runtimes.**
    - The data comes from `simctl runtime list -j` and `simctl list devices -j`: id, platform, version, `sizeBytes`, `path`, `parentMountPath`, `deletable`, `state`, devices on the runtime, and whether any of them is booted. The fields were read from Xcode 27's `simctl` on 2026-10-28.
    - The area's bytes come from walking AssetsV2; the runtimes' come from `simctl`. When they differ, the row shows both: "listed runtimes X GB; Y GB kept by macOS". Apple's forums report runtime assets that `simctl` no longer lists and that no tool can remove under SIP.
    - `xcrun` on a Mac without developer tools opens the Command Line Tools dialog, so it is never called. `xcode-select -p` must succeed and `xcodebuild` must exist under it.
    - The `simctl` under `xcode-select -p` is a shell shim. When CoreSimulator is missing or older than the shim expects, which is normal right after an Xcode update, the shim runs `xcodebuild -runFirstLaunch`, which installs components and needs an administrator. So `ddc` never runs the shim. It first runs `xcodebuild -checkFirstLaunchStatus`, and only when that exits 0 does it run the real binary, `/Library/Developer/PrivateFrameworks/CoreSimulator.framework/Versions/A/Resources/bin/simctl`. Otherwise the row says "not measured until Xcode finishes its first launch". Checked on the reference machine on 2026-10-28: `xcodebuild -checkFirstLaunchStatus` printed nothing, exited 0 and showed no dialog.
    - The runtimes are joined to devices on `runtimeIdentifier` (`com.apple.CoreSimulator.SimRuntime.…`), the key `list devices -j` groups devices under. `identifier` is the id `runtime delete` takes.
    - Without developer tools the row is absent.
  - **Rows without a tool:**
    - swap: the VM volume, which a restart frees;
    - `/private/var/vm/sleepimage`;
    - a downloaded macOS update in `/Library/Updates`, freed by installing it;
    - conda, freed by `conda clean`;
    - `/usr/local`;
    - `/Users/Shared`;
    - other users' home folders: the `Users` row, with what `Users/Shared` does not hold; what cannot be read is "not counted";
    - local Time Machine snapshots: their count from `tmutil listlocalsnapshots /`. Their size cannot be measured without root, so the row says so.
- [x] **"Not counted"** is the Data volume's consumed bytes minus everything counted on it.
  - **It is never clamped.** When the counted bytes exceed the volume's, the report sets `inconsistent: true` and names the likely cause: a stale home row, or clones counted on both sides.
  - Its explanation lists what it holds:
    - the Spotlight index, `.fseventsd` and `.DocumentRevisions-V100`;
    - folders readable only by root;
    - APFS metadata;
    - the named rows of the home walk above.
- [x] **Report and running.**
  - **Saving.** The report is written atomically to `<data-dir>/last-disk.json`, with the schema `docs/disk.schema.json` at version 1.
  - **Lock.** `ddc disk` takes no ddc lock: it reads nothing that a clean writes, except `last-scan.json`, which it only reads. Two writers can overlap: a `ddc disk` from Terminal and one from the app, or a Terminal `ddc disk` that spans a `disk free`. The last one to finish writes `last-disk.json`, possibly with Homebrew bytes from before a cleanup and without the stale mark, and that is accepted; the next `ddc disk` corrects it.
  - **Progress** is `--progress json`, as for `scan`. The progress schema's `WalkStep` is a closed set (`scan`, `recheck`, `markers`), so `disk` adds a variant, and the schema and its contract test change with it. The app's `ProgressEvent.WalkStep` gains `case disk`, the one line this story changes in `app/`, because `ContractTests` compares that enum with the schema. `PROGRESS_SCHEMA_VERSION` is not bumped: a bump would make the app drop every progress event, while the app already ignores a `walk` line whose step it does not know.
  - **Stopping.** SIGTERM stops the walk. The report is still written, with `cancelled: true` and the unfinished areas marked partial.
  - **Text output.** Without `--json` it prints a table of the areas.

## Implementation Plan

Planned on 2026-10-28 against the code at `aa64c83`.

1. **Ports.**
   - `sys/statfs.rs`: `VolumeStats` gains `free_bytes` (`f_bfree × f_bsize`) and `mounted_from` (`f_mntfromname`), and `mounts()` lists the mounted file systems through `getfsstat`. The `StatFs` port gains `mounts()` with a default that returns `Unsupported`, so the existing fakes still compile.
   - `exec`: `CommandRunner::at(exe)` returns an absolute path when it is an executable file. It shares one check with `SystemCommands::find`, and the fake commands port answers by file name.
   - `exec::count_local_snapshots(runner, exe) -> Option<usize>`: `None` means unknown. `local_snapshots` becomes a wrapper around it.
2. **One promise not to open.** The skip check in `RuleVisitor::before_read` moves into `rules::NeverOpen` (the user's skips and the catalog's no-traverse paths). The `RuleVisitor` and the disk visitor both call it. The existing scan, report and settings tests stay green. `Settings::resolved_skips` gives the skips without requiring the scan roots to exist.
3. **The report, `ddc-core/src/disk/`.**
   - `model.rs`: `DiskReport`, `DISK_SCHEMA_VERSION = 1`, kebab-case enums, and the action ids as a closed set (`simctl-runtime-delete`, `brew-cleanup`).
   - `areas.rs`: the fixed table. The walk roots are top-level Data folders (`Applications`, `Library`, `opt`, `private`, `System/Library/AssetsV2`, `usr/local`, `Users`), and nested areas (`Library/Updates`, `private/var/folders`, `private/tmp`, `private/var/vm`, `opt/homebrew`, `Users/Shared`) are regions inside them. A missing root is left out, and roots that are the same folder by `(dev, ino)` are walked once. Areas are disjoint: the walker credits a directory to its innermost region, so a nested area's bytes leave its parent's (`Library` is `/Library` less `Library/Updates`), and the rows plus the home row sum to the walk's total. The `Users` row holds other users' home folders and the rest of `Users`; `/private/var/vm/sleepimage` is the `private/var/vm` region.
   - `visitor.rs`: `DiskVisitor` asks `NeverOpen` with the walked path and, under `/System/Volumes/Data`, with that prefix stripped, so a skip outside the home folder (`/opt/x`) is honoured too. It tags the regions and counts entries toward the temporary-folders budget. It does not open a directory of a budgeted area with more entries than the budget has left, or than 10,000, and the area is then partial. The entries come from one `lstat`: on APFS the link count is the entries plus two, up to 65,535, where it stops; past it, the size is 32 bytes per entry, plus two. On another file system the link count is lower (HFS+ counts subdirectories only), so the directory is read: the guard fails open. On the reference machine on 2026-10-28, `$TMPDIR` held about 838,000 entries (link count 65,535, size 26.8 MB). Its one listing took about 300 s before any entry counted, while the other 2.56 million files took 15 s; with the guard, `ddc disk` took 13 s. It prunes the user's own home inside `Users`, and prunes everything once the signal flag (the `AtomicBool` of `signals::install()`) is set. It is not `Progress::cancel`: the walker checks that before it asks `prunes`, so no area would be marked partial. Each prune marks its area partial.
   - `apfs.rs`: `diskutil apfs list -plist` through a small XML plist parser of its own (no new dependency). The container is the one holding the volume that `statfs(/System/Volumes/Data)` is mounted from (`disk3s5`), which leaves out the cryptex and disk-image containers. The fallback is `statfs` per mounted APFS volume whose `mounted_from` shares the Data volume's `diskN` prefix, with Recovery reported as not measured.
   - `home.rs`: the last scan is used only when it has the current schema, `scan_roots` is exactly the home folder, a volume entry puts the home on the Data mount, it was not cancelled, and it is at most 24 hours old. Otherwise the home folder is one more walk root, with its own region, given as `env.home()` (`/Users/<name>`; `canonicalize` does not rewrite a firmlink). Under `/System/Volumes/Data/Users/<name>` the walker would drop it as nested in the `Users` root, and `NeverOpen`, keyed by the catalog's `home.join(…)` and the canonical skips, would not fire. The `Users` root hides the same folder by `(dev, ino)`.
   - `simulators.rs`: `xcode-select -p`, then `xcodebuild` under it, then `-checkFirstLaunchStatus`, then the CoreSimulator `simctl`. JSON is read leniently, and runtimes are joined to devices on `runtimeIdentifier`.
   - "Not counted" is the Data volume's consumed bytes minus the home row plus the walk's total, signed and never clamped. When negative, `inconsistent: true` names `stale-home-row` when the home row's source is `last-scan`, else `clones-counted-twice`.
4. **Store and progress.** `store::write_last_disk` writes `last-disk.json` atomically, with no lock. `WalkStep::Disk` is added, `docs/progress.schema.json` is regenerated, and the Swift enum gains `case disk`.
5. **CLI.** `ddc disk [--json] [--progress FORMAT] [--data-dir] [--threads]`, with a hidden `--system-root DIR` for tests. `--system-root DIR` replaces `/` for the walk roots, for `statfs(DIR/System/Volumes/Data)`, and for the tool paths (`usr/sbin/diskutil`, `usr/bin/tmutil`, `usr/bin/xcode-select`, the CoreSimulator `simctl`); that is how the CLI tests substitute their fakes. It installs the signal handler first, then measures in the order container, snapshots, simulators, walk, skipping each step once the signal flag is set, so a SIGTERM during the fake `diskutil` leaves every area partial. The final `walk` tick carries the report's walk totals. Each tool runs under a timeout (`diskutil` 30 s, the developer tools 60 s). It writes `last-disk.json` even when cancelled, prints JSON or a table, then re-raises a received signal. The README's command-line section documents it.
6. **Tests.** `ddc-core/tests/disk.rs` covers the criteria about areas, the home source, "not counted", no findings, the budget, developer tools, a core cancel and never opening, including a skip outside the home folder named through `/` that the walk reaches through the Data volume. A fixture can only make a firmlink-like path with a symlink, which `canonicalize` collapses, so the `(dev, ino)` dedup is shown by an area reached through a symlink. `ddc-cli/tests/disk_cli.rs` covers JSON, progress and SIGTERM, with a fake `diskutil` held at a gate. The schema test generates `docs/disk.schema.json`, and a structure test keeps `disk` away from the cleanup code (`Executor`, `plan::`, `trash`, `xcrun`).
7. **The `diskutil` fixture** is a capture from the reference machine with the UUIDs and physical stores removed.

## Acceptance Criteria

- [x] Fixture test: every area of a fixture table is reported with its bytes. A fixture folder reachable through both a firmlink-like path and its Data path is counted once. — evidence: `crates/ddc-core/tests/disk.rs::every_area_is_reported_with_its_bytes_and_a_folder_reached_twice_counts_once`, `::an_area_that_is_another_area_by_dev_and_ino_is_measured_once`
- [x] Test: the home row comes from `last-scan.json` only when it is whole-home, of the current schema, on the Data volume and at most 24 hours old. A report of `~/Code` only, a report of a root on another volume, and a two-day-old report each make `disk` walk the home folder itself. The row says which source it used. — evidence: `crates/ddc-core/tests/disk.rs::the_home_row_comes_from_the_last_scan_only_when_it_still_describes_the_whole_home` (schema, cancelled, `~/Code`, another volume, two days)
- [x] Test: "not counted" equals consumed minus counted. When the counted bytes exceed consumed, the report is `inconsistent: true` with a cause, and no value is clamped. — evidence: `crates/ddc-core/tests/disk.rs::not_counted_is_consumed_minus_counted_and_never_clamped`
- [x] Test: the size-only walk produces no findings. A `__pycache__` or `node_modules` inside an area appears only in the area's bytes. — evidence: `crates/ddc-core/tests/disk.rs::the_size_only_walk_produces_no_findings`, `crates/ddc-cli/tests/structure.rs::the_disk_report_never_reaches_cleanup`
- [x] Test: the temporary-folders budget stops the walk of a fixture with more entries than the budget, and the row is partial with the entries read. — evidence: `crates/ddc-core/tests/disk.rs::the_entry_budget_stops_the_temporary_folders_and_marks_them_partial`, `::a_folder_with_more_entries_than_the_budget_has_left_is_never_opened`
- [x] Test: without a developer directory (fake `xcode-select` failing), no `simctl` or `xcrun` is run, and the runtime row is absent. With a fake `xcodebuild -checkFirstLaunchStatus` reporting a pending first launch, no `simctl` is run and the row says so. With a fake `simctl`, runtimes are reported with their devices and a booted flag. — evidence: `crates/ddc-core/tests/disk.rs::simulator_runtimes_need_developer_tools_and_a_finished_first_launch`
- [x] Test: SIGTERM during the walk writes `last-disk.json` with `cancelled: true` and partial areas. — evidence: `crates/ddc-cli/tests/disk_cli.rs::sigterm_while_diskutil_runs_ends_it_and_still_saves_a_partial_report`, `crates/ddc-core/tests/disk.rs::a_stop_during_the_walk_still_reports_with_the_unfinished_areas_partial`
- [x] `docs/disk.schema.json` is generated like the other schemas and checked by the schema test. — evidence: `crates/ddc-core/tests/disk.rs::committed_disk_schema_is_current`
- [x] Test: the home fallback walk never opens a folder in `skips` or a cloud folder. A fixture with one of each, and a reader that records opens, shows no open below either. — evidence: `crates/ddc-core/tests/disk.rs::the_home_walk_never_opens_a_skip_or_a_cloud_folder`, `::a_skip_outside_the_home_folder_is_never_opened_through_the_data_volume`
- Manual checks (1) moved to [Owner checks: manual acceptance across stories](../../../ops/owner-checks-manual-acceptance-across-stories.md) on 2026-10-30, by the owner's decision.

## Final Summary

**What changed** (code commit 62602bd, released in 0.4.0 with c164663):
- `ddc disk [--json]`: a read-only report of the startup container as areas with sizes, from a size-only walk outside the home folder. It makes no findings and removes nothing.
- The home row comes from `last-scan.json` only when it is whole-home, of the current schema, on the Data volume and at most 24 hours old. Otherwise `disk` walks the home itself and never opens a skip or a cloud folder.
- "Not counted" is consumed minus counted, signed and never clamped; a negative value marks the report `inconsistent` with a cause.
- Simulator runtimes are read through `simctl` only with developer tools present and the first launch done. The temporary folders have an entry budget. SIGTERM still saves a partial `last-disk.json`. `docs/disk.schema.json` is generated.

**Tests run:** `crates/ddc-core/tests/disk.rs`, `crates/ddc-cli/tests/disk_cli.rs`, and `crates/ddc-cli/tests/structure.rs::the_disk_report_never_reaches_cleanup`.

**Remaining risk:** one manual check moved to the owner list: the run time (under 3 minutes) and the area sizes against `du` and the Data volume's consumed bytes on the reference machine. Until it runs, the accuracy on a real disk is not checked.

## Technical Notes

- The area table is code, not settings. The titles and explanations are what the app shows.
- The signal handler's notice, "stopping after the current item", is shared with `clean` and stays as it is for `disk`: the current item is the folder being read.
- A listing of hundreds of thousands of entries stalls any walk, not only in the temporary folders. The 10,000-entry limit applies only to budgeted areas, because elsewhere it would make a whole row partial; one `lstat` per directory could extend it to every area if such a folder turns up.
- Where the code differs from the plan (reviewer, 2026-10-28): the skips are resolved through `resolve` with the home folder as the only path, not a new `Settings::resolved_skips`; the walk root is all of `System` (its remainder is the `macos-assets` row), and `opt`, `private`, `Users`, `Library` and `cores` are rows for what their nested areas leave, so the rows plus the home row add up to the walk; the Swift side gains `case disk` and one more pattern in each of two exhaustive switches (`CleanTimings`, `CleanProgress`), with no new text. When two area paths are one folder by `(dev, ino)`, the first in the table keeps it, roots before nested areas.
- `/usr/share/firmlinks` lists the firmlinked folders. They are counted through `/System/Volumes/Data` only.
- The cryptex mounts of simulator runtimes under `/private/var/run/com.apple.security.cryptexd/mnt` are other file systems. Their images are counted once, in `/System/Library/AssetsV2`.

## Dependencies

- [[research/what-fills-system-data-on-a-developers-mac|What fills System Data on a developer's Mac]]
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]: unchanged by this story; the report removes nothing
- [[adr/rust-core-library-and-cli-with-json-output|Rust core: library + CLI with JSON output]]: amended, `getfsstat` joins the `unsafe` list
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]: amended, §6 walk steps gain `disk`

## Attachments

---
type: story
id: "story-whole-disk-what-lies-outside-the-home-folder"
epic: "DDC-002"
title: "Whole disk: what lies outside the home folder"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [cli, whole-disk, system-data]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Whole disk: what lies outside the home folder

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
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

- [ ] **The startup container and its volumes.**
  - The source is `/usr/sbin/diskutil apfs list -plist`. System tools are run by absolute path (`/usr/sbin/diskutil`, `/usr/bin/tmutil`, `/usr/bin/xcode-select`), because `SystemCommands::find` searches only the tool directories and `/usr/bin`. The plist is filtered to the container whose volume is mounted at `/System/Volumes/Data`. The plist also lists the simulator runtimes' cryptex containers and mounted disk images, which are not part of it.
  - For each volume it gives the role, the bytes consumed and whether it is mounted, plus the container's "in use" and capacity.
  - If `diskutil` fails, the fallback is `statfs` per mounted volume. Used is `f_blocks − f_bfree`; the port gains `f_bfree`. Recovery is then reported as not measured.
- [ ] **The home folder row.**
  - **When `last-scan.json` can be used**, its bytes are taken as they are. The report must have the current schema, its `scan_roots` must be exactly the home folder, it must lie on the Data volume, and it must be at most 24 hours old.
  - **Otherwise `ddc disk` sizes the home folder itself.** The walk is size-only, but it keeps what the scan never opens: the cloud folders (today the catalog's `NoTraverse` matcher) and the user's `skips` (today in the `RuleVisitor`). Neither is in `WalkOptions`, so the plan gate chooses between a size-only visitor that keeps both and moving both into `WalkOptions`. A plain `NullVisitor` walk of the home folder is not allowed, because it would open the folders the safety ADR promises are never opened.
  - **The row's date** is the report's `generated_at`. After a clean that edited `last-scan.json` in place, the bytes can be newer than that date; the row says "from the scan at …" anyway, which errs on the old side.
  - **The row carries its source and date.** The source is `last-scan` or `walk`.
  - **What the home walk does not read becomes named rows inside "not counted", never silent:**
    - the cloud folders, `~/Library/CloudStorage` and `~/Library/Mobile Documents`;
    - the user's `skips`;
    - unreadable folders, with their count.
- [ ] **Areas outside the home folder.** A fixed table in code gives each area its paths, title, explanation and optional action id. All paths go through `/System/Volumes/Data/...`, because the firmlinks would otherwise count `/Applications`, `/Library`, `/opt`, `/private`, `/usr/local` and `/System/Library/AssetsV2` twice.
  - **The size-only walk** is the one `ddc du` already uses (`NullVisitor`). Each inode is counted once, unreadable folders and other file systems are recorded, and nothing is classified.
  - **Temporary folders** (`/private/var/folders`, `/private/tmp`) are walked under a budget of entries. Past the budget the row is marked partial ("at least"), with the entries read. On the reference machine one `$TMPDIR` holds about 10 million directories, which made a whole-disk walk take 10.5 minutes.
  - **Simulator runtimes.**
    - The data comes from `simctl runtime list -j` and `simctl list devices -j`: id, platform, version, `sizeBytes`, `path`, `parentMountPath`, `deletable`, `state`, devices on the runtime, and whether any of them is booted. The fields were read from Xcode 27's `simctl` on 2026-10-28.
    - The area's bytes come from walking AssetsV2; the runtimes' come from `simctl`. When they differ, the row shows both: "listed runtimes X GB; Y GB kept by macOS". Apple's forums report runtime assets that `simctl` no longer lists and that no tool can remove under SIP.
    - `xcrun` on a Mac without developer tools opens the Command Line Tools dialog, so it is never called. `xcode-select -p` must succeed and `xcodebuild` must exist under it.
    - The `simctl` under `xcode-select -p` is a shell shim. When CoreSimulator is missing or older than the shim expects, which is normal right after an Xcode update, the shim runs `xcodebuild -runFirstLaunch`, which installs components and needs an administrator. So `ddc` never runs the shim. It first runs `xcodebuild -checkFirstLaunchStatus`, and only when that exits 0 does it run the real binary, `/Library/Developer/PrivateFrameworks/CoreSimulator.framework/Versions/A/Resources/bin/simctl`. Otherwise the row says "not measured until Xcode finishes its first launch". The plan gate confirms that the status check shows no dialog.
    - The runtimes are joined to devices on `runtimeIdentifier` (`com.apple.CoreSimulator.SimRuntime.…`), the key `list devices -j` groups devices under. `identifier` is the id `runtime delete` takes.
    - Without developer tools the row is absent.
  - **Rows without a tool:**
    - swap: the VM volume, which a restart frees;
    - `/private/var/vm/sleepimage`;
    - a downloaded macOS update in `/Library/Updates`, freed by installing it;
    - conda, freed by `conda clean`;
    - `/usr/local`;
    - `/Users/Shared`;
    - other users' home folders: their size if readable, else "not counted";
    - local Time Machine snapshots: their count from `tmutil listlocalsnapshots /`. Their size cannot be measured without root, so the row says so.
- [ ] **"Not counted"** is the Data volume's consumed bytes minus everything counted on it.
  - **It is never clamped.** When the counted bytes exceed the volume's, the report sets `inconsistent: true` and names the likely cause: a stale home row, or clones counted on both sides.
  - Its explanation lists what it holds:
    - the Spotlight index, `.fseventsd` and `.DocumentRevisions-V100`;
    - folders readable only by root;
    - APFS metadata;
    - the named rows of the home walk above.
- [ ] **Report and running.**
  - **Saving.** The report is written atomically to `<data-dir>/last-disk.json`, with the schema `docs/disk.schema.json` at version 1.
  - **Lock.** `ddc disk` takes no ddc lock: it reads nothing that a clean writes, except `last-scan.json`, which it only reads. Two writers can overlap: a `ddc disk` from Terminal and one from the app, or a Terminal `ddc disk` that spans a `disk free`. The last one to finish writes `last-disk.json`, possibly with Homebrew bytes from before a cleanup and without the stale mark, and that is accepted; the next `ddc disk` corrects it.
  - **Progress** is `--progress json`, as for `scan`. The progress schema's `WalkStep` is a closed set (`scan`, `recheck`, `markers`), so `disk` adds a variant, and the schema and its contract test change with it.
  - **Stopping.** SIGTERM stops the walk. The report is still written, with `cancelled: true` and the unfinished areas marked partial.
  - **Text output.** Without `--json` it prints a table of the areas.

## Implementation Plan

## Acceptance Criteria

- [ ] Fixture test: every area of a fixture table is reported with its bytes. A fixture folder reachable through both a firmlink-like path and its Data path is counted once.
- [ ] Test: the home row comes from `last-scan.json` only when it is whole-home, of the current schema, on the Data volume and at most 24 hours old. A report of `~/Code` only, a report of a root on another volume, and a two-day-old report each make `disk` walk the home folder itself. The row says which source it used.
- [ ] Test: "not counted" equals consumed minus counted. When the counted bytes exceed consumed, the report is `inconsistent: true` with a cause, and no value is clamped.
- [ ] Test: the size-only walk produces no findings. A `__pycache__` or `node_modules` inside an area appears only in the area's bytes.
- [ ] Test: the temporary-folders budget stops the walk of a fixture with more entries than the budget, and the row is partial with the entries read.
- [ ] Test: without a developer directory (fake `xcode-select` failing), no `simctl` or `xcrun` is run, and the runtime row is absent. With a fake `xcodebuild -checkFirstLaunchStatus` reporting a pending first launch, no `simctl` is run and the row says so. With a fake `simctl`, runtimes are reported with their devices and a booted flag.
- [ ] Test: SIGTERM during the walk writes `last-disk.json` with `cancelled: true` and partial areas.
- [ ] `docs/disk.schema.json` is generated like the other schemas and checked by the schema test.
- [ ] Test: the home fallback walk never opens a folder in `skips` or a cloud folder. A fixture with one of each, and a reader that records opens, shows no open below either.
- [ ] Manual (owner): on the reference machine, `ddc disk` finishes in under 3 minutes. Each named area is within 5% of a `du -skx` of the same paths taken the same day, and "not counted" is within 10 GB of the Data volume's consumed bytes minus everything counted.

## Final Summary

## Technical Notes

- The area table is code, not settings. The titles and explanations are what the app shows.
- `/usr/share/firmlinks` lists the firmlinked folders. They are counted through `/System/Volumes/Data` only.
- The cryptex mounts of simulator runtimes under `/private/var/run/com.apple.security.cryptexd/mnt` are other file systems. Their images are counted once, in `/System/Library/AssetsV2`.

## Dependencies

- [[research/what-fills-system-data-on-a-developers-mac|What fills System Data on a developer's Mac]]
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]: unchanged by this story; the report removes nothing

## Attachments

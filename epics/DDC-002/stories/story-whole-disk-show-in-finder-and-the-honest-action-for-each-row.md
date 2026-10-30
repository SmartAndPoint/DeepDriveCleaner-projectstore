---
type: story
id: "story-whole-disk-show-in-finder-and-the-honest-action-for-each-row"
epic: "DDC-002"
title: "Whole disk: Show in Finder and the honest action for each row"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-29
updated: 2026-10-30
external_refs: {}
tags: [core, app, whole-disk, finder, user-report]
code_refs: ["crates/ddc-core/src/disk/model.rs", "crates/ddc-core/src/disk/areas.rs", "crates/ddc-core/src/disk/mod.rs", "crates/ddc-core/tests/disk.rs", "docs/disk.schema.json", "bench/golden/system-root/disk.json", "app/Packages/DDCKit/Sources/DDCKit/DiskReport.swift", "app/Packages/DDCKit/Sources/DDCKit/AreaActions.swift", "app/Packages/DDCKit/Sources/DDCKit/DiskSummary.swift", "app/Packages/DDCKit/Tests/DDCKitTests/RevealTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ContractTests.swift", "app/DeepDriveCleaner/RevealButton.swift", "app/DeepDriveCleaner/DiskView.swift", "app/DeepDriveCleaner/AgentsSection.swift", "app/Packages/DDCKit/Sources/DDCKit/AgentsSummary.swift", "app/Packages/DDCKit/Tests/DDCKitTests/AgentsTests.swift"]
specs: []
started_at: "2026-10-29T07:01:39.440Z"
closed_at: "2026-10-30T14:28:50.764Z"
plan_updated_at: "2026-10-29T07:01:39.440Z"
---

# Whole disk: Show in Finder and the honest action for each row

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner, 2026-10-29: the Whole disk rows (Applications, Library for all users, Conda, /usr/local, System files in /private) say what frees their space but offer nothing to press. Where DeepDriveCleaner cannot act itself, it should send the user to Finder or to the owning program. The analysis, with the table below, is in the research note "Removal wherever a size is shown: coding agents, protected folders and the whole disk" (Finding 4).

DeepDriveCleaner removes nothing by path outside the home folder (safety ADR amendment of 2026-10-28), and hands the user no `sudo` command. So each row gets the honest action it has, and no more. Conda and Homebrew get their tools in their own stories; Show in Finder does not wait for them.

| Row (`disk/areas.rs` id) | Action |
|---|---|
| Applications (`applications`) | Show in Finder |
| Library for all users (`library`) | Show in Finder |
| macOS downloaded data (`macos-assets`) | — |
| /opt (`opt`) | Show in Finder |
| /usr/local (`usr-local`) | Show in Finder |
| System files in /private (`private`) | — |
| Other users (`users`) | — |
| Crash dumps (`cores`) | Show in Finder. Finder asks for an administrator's password itself when the user deletes there |
| Downloaded macOS update (`macos-update`) | Open Software Update (the System Settings pane) |
| Simulator runtimes, Homebrew, Conda | Their tool sheets, plus Show in Finder |
| Temporary folders (`temporary-folders`), /tmp (`tmp`), Sleep image (`sleep-image`) | — |
| Shared folder (`shared`) | Show in Finder |

- **Where the action lives.** In the core's area table, as a new field `reveal` beside `frees` (the existing `action` stays the tool action): an enum `AreaReveal` with `finder` and `software-update`, or none. The app shows the button from the field. A row with no action keeps its explanation and gets no button. Every row also gets a context menu with the same action.
- **Show in Finder** selects the area's folder (`NSWorkspace.activateFileViewerSelecting`). A row with several paths (Conda) selects the ones that exist. The disk report spells areas through `/System/Volumes/Data/...` (safety ADR amendment); Show in Finder reveals the firmlinked path the user knows (`/opt/miniconda3`, `/Applications`), or Finder opens inside the Data volume.
- **Open Software Update** opens `x-apple.systempreferences:com.apple.Software-Update-Settings.extension`. It shows no dialog of ours.
- **No Terminal button.** A `.command` file runs the moment Terminal opens, with no pause to read it. Where a command is the action, the tool sheet shows it with Copy Command, as it does today.
- **The Coding Agents section** (story "Coding agents in the Whole disk view") gets Show in Finder on each harness's folders. It stays non-destructive, so `ContractTests.agentsCallers` holds.

## Decomposition

- [x] Core: the action field in the area table and in `disk --json`; the schema regenerated.
- [x] DDCKit: decode the field; which rows show which button.
- [x] App: the buttons and the context menus on the rows, and Show in Finder in the Coding Agents section. (Whole disk rows in commit 2af34f0; the Coding Agents section in PR #5, c9b84dc)

## Implementation Plan

Planner (2026-10-29). No covering spec; the Description is the source. Only Decomposition item 3 (the Coding Agents section) waits for "Coding agents in the Whole disk view" (DDC-002); steps 1–5 do not.

1. Core, `crates/ddc-core/src/disk/model.rs`: `AreaReveal` (kebab-case: `finder`, `software-update`) beside `ActionId`; `Area.reveal: Option<AreaReveal>` with `serde(default, skip_serializing_if)`. No `DISK_SCHEMA_VERSION` bump: the field is optional and the app decodes it leniently; its doc comment names `AreaReveal` as the exception to "a new variant needs a bump".
2. Core, `disk/areas.rs`: `AreaDef.reveal`, default `None` in `area()`. `finder`: applications, library, opt, usr-local, cores, simulator-runtimes, homebrew, conda, shared. `software-update`: macos-update. None: macos-assets, private, users, temporary-folders, tmp, sleep-image. `disk/mod.rs`: `reveal: def.reveal` in the one `Area` literal. The `cores` "frees" text loses "in Terminal": "Delete them in Finder; it asks for an administrator's password."
3. Regenerate `docs/disk.schema.json` (`DDC_UPDATE_SCHEMA=1`) and the golden `bench/golden/system-root/disk.json` (`make golden-record`); review both diffs.
4. DDCKit, `DiskReport.swift`: `AreaReveal` (String, CaseIterable, not Decodable); `Area.reveal` through a hand-written `init(from:)` as `NotCounted` has, so an unknown value or a missing key is nil. `AreaActions.swift`: `Reveal` (`.finder([String])`, `.softwareUpdate`) with its title, `reveal(_ area:)`, `userPath` (strips `/System/Volumes/Data` only at a path boundary), `existing(_:exists:)`, `softwareUpdateURL`. `DiskSummary.Row.reveal`, set only for area rows.
5. App: `RevealButton.swift` shaped like `OpenTrashButton.swift` (`activateFileViewerSelecting` on the paths that exist now, disabled when none; `NSWorkspace.open` for Software Update). `DiskView.swift` `DiskRow`: the button before the tool's offer, and a context menu with it.
6. After the agents story closes: `AgentsSummary.Row` paths (folder: its path; harness: its top-level folders; sessions/transcripts/records: none) and a Show in Finder context menu in `AgentsSection.swift`; `agentsCallers` unchanged and green.
7. Tests: core `every_area_has_the_reveal_of_the_story_table` (all ids, id sets equal), `the_report_carries_each_areas_reveal`, schema enum; DDCKit `RevealTests` (finder/software-update/nil rows, titles, URL, unknown value and missing key nil, `userPath` both shapes and `DataX`, `existing`), `diskContracts` gains `AreaReveal`. Fixture `disk-report.json` gets only valid values (it round-trips through Rust). Then `swift test`, `make gate`, reviewer.

Pitfalls: DDCKit fixtures use `/Applications`, real reports `/System/Volumes/Data/Applications`, so `userPath` must leave an unprefixed path alone; a report saved before this change shows no reveal buttons until the next whole-disk scan; Finder may not show a selected dot-folder while hidden files are hidden (owner's check); no Terminal action, `.command` file or `sudo` text anywhere.

## Acceptance Criteria

- [x] Test (core): every row in `disk/areas.rs` has the `reveal` of the table above; the JSON carries it, and the schema names the enum `AreaReveal`. — evidence: `crates/ddc-core/tests/disk.rs` `every_area_has_the_reveal_of_the_story_table`, `the_report_carries_each_areas_reveal`, `the_disk_schema_names_area_reveal`, `committed_disk_schema_is_current` (commit 2af34f0)
- [x] Test (DDCKit): a row with `finder` shows Show in Finder; with `software-update` shows Open Software Update; with none shows no button; an unknown value decodes as none. — evidence: `RevealTests.swift` `rowsFromTheReport`, `softwareUpdate`, `lenient`, `userPath`, `existing`; `ContractTests.diskContracts` AreaReveal (`swift test` 285 passed)
- [x] Test (DDCKit): `ContractTests.agentsCallers` still finds no destructive call in the Coding Agents section. — evidence: `ContractTests.agentsCallers` (`ContractTests.swift:456-476`, PR #5): still no destructive call in `AgentsSection.swift`, and now exactly two `Button(`/`RevealButton(` calls (Refresh, Show in Finder), `RevealButton(reveal: .finder(` present, no `NSWorkspace`, `softwareUpdate` or `recycle`; `AgentsSummaryTests.finderPaths`; Gate run 41068463547, job `app`, 385 tests passed; unsigned Debug app build passed before the PR
- [ ] Manual (owner): on the Whole disk view, each row's button or its absence matches the table; Show in Finder opens the right folder at its usual path, not under `/System/Volumes/Data`. — moved to the runbook "Owner checks: manual acceptance across stories", with the Coding Agents section's dot-folders added

## Final Summary

- Done so far (commit 2af34f0): the area table carries `reveal` (`finder`, `software-update`, none) into `disk --json` and `docs/disk.schema.json`, with no schema bump. DDCKit reads it leniently: an unknown value means no button. The Whole disk rows show Show in Finder or Open Software Update beside the tool's offer, with the same action in a context menu. Show in Finder selects the folders that exist now, by their firmlinked paths. The crash dumps row now says to delete in Finder, which asks for the password.
- Tests: `cargo nextest -p ddc-core --test disk` 20 passed; golden re-recorded (adds only `reveal` and the cores wording); `swift test` 285 passed; unsigned Debug app build passed. The reviewer found ACs 1–2 met and no blockers.
- Coding Agents section (PR #5, c9b84dc): `AgentsSummary.Row.paths` gives a folder its own path and a harness its top-level folders (a nested folder is never selected twice); sessions, records and transcripts have none and get no menu. `AgentsRow` adds a Show in Finder context menu through `RevealButton` when there are paths. `agentsCallers` was tightened to pin the section to its two buttons. `swift test` 380 passed, unsigned app build passed, Gate 41067221767 green; shipped in 0.9.0.
- Open for the owner: AC 4, in the runbook "Owner checks: manual acceptance across stories", which also asks whether Finder visibly selects a hidden harness folder such as `~/.claude`. The code landed before "Coding agents in the Whole disk view" closed, against the order in Dependencies; both close together. A report saved before this change shows no reveal buttons until the next whole-disk scan. Nits left: the empty context menu on rows with no action, and VoiceOver with two buttons in a combined row, both for the owner's check.

## Technical Notes

- Show in Finder needs no permission; it reveals, it does not read.

## Dependencies

- Research note "Removal wherever a size is shown: coding agents, protected folders and the whole disk" (Finding 4)
- "Coding agents in the Whole disk view" (DDC-002): the section and its contract test; this story waits for it to close

## Attachments

-

---

*Last updated: 2026-10-30*

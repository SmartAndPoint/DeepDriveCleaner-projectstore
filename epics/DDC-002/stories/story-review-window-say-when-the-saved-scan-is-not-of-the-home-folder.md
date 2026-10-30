---
type: story
id: "story-review-window-say-when-the-saved-scan-is-not-of-the-home-folder"
epic: "DDC-002"
title: "Review window: say when the saved scan is not of the home folder"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-29
updated: 2026-10-30
external_refs: {}
tags: [app, review, popover, cli, defect, user-report]
code_refs: ["app/Packages/DDCKit/Sources/DDCKit/ReportScope.swift", "app/Packages/DDCKit/Sources/DDCKit/Formatting.swift", "app/Packages/DDCKit/Sources/DDCKit/PopoverModel.swift", "app/Packages/DDCKit/Sources/DDCKit/ReviewModel.swift", "app/DeepDriveCleaner/ReviewView.swift", "app/DeepDriveCleaner/PopoverView.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ReportScopeTests.swift", "crates/ddc-core/src/env.rs", "crates/ddc-cli/src/cmd/scan.rs", "crates/ddc-cli/tests/scan_cli.rs"]
specs: []
started_at: "2026-10-29T06:40:44.528Z"
closed_at: "2026-10-30T06:45:42.874Z"
plan_updated_at: "2026-10-29T06:40:44.528Z"
---

# Review window: say when the saved scan is not of the home folder

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

A defect found by the owner on 2026-10-29 (app 0.4.1). The Review window showed 74 temp folders under "Protected by you". The saved report was not of the home folder: it came from a manual `ddc scan /private`, which saves to the app's data folder like any scan. The Review window showed it as the home folder, without a word, and offered Clean and Protect on it. The analysis is in the research note "Removal wherever a size is shown: coding agents, protected folders and the whole disk" (Finding 1).

**What already checks the roots.** The popover's `Wording.scope(roots:home:)` (`Formatting.swift`) says "found in private". In the core, `disk/home.rs` refuses a non-home report for the Whole disk home row, `report/recheck.rs::same_roots` gates the clean's report edits, and the plan refuses items outside the current roots. The Review window has no check. "found in private" does not explain what happened.

**The app's roots, computed as the CLI computes them** (`settings.rs`, `scan/walker.rs`):

- the Settings roots as the whole list, or the home when the list is empty;
- without the roots that do not exist now (an unplugged disk), as the CLI skips them;
- each through `realpath(3)`. Foundation's `resolvingSymlinksInPath` strips a leading `/private` and would never equal Rust's `canonicalize`;
- sorted, deduplicated, nested roots dropped.

One DDCKit helper computes this list. It compares it with `report.scanRoots`, and the app's home with `report.home`, both through `realpath(3)` (`same_roots` in `report/recheck.rs` compares the home raw; `disk/home.rs` is the one that canonicalises). A report whose roots match but whose home differs (made under another user) is "outside". `PopoverModel` and `ReviewModel` both use it. A cancelled report keeps today's "incomplete" wording and is not part of this story, nor is an age limit.

**Two cases.** A report root that is a Settings root unplugged now is left out of the comparison, so a report made while that disk was plugged in still counts as the same list, with no notice. An unplugged root cannot go through `realpath(3)`, so this match compares the Settings entry as written with the report's canonical root, as strings. (The CLI's `same_roots` would call such a report foreign and take a full walk; the notice is what differs here, and nothing is removed on either verdict.) Otherwise:

- **A report root lies under none of the Settings roots, plugged in or not** (the `/private` case, or an external disk scanned from the terminal). The Review window and the popover say: "This report is of /private, not of your home folder and the folders in Settings. Scan Now replaces it." With several roots: "This report is of 2 folders, not of …", counted as `Wording.scope` counts them. The Scan Now button sits beside the text. Clean and Protect are off, because Clean would end in refusals and Protect would write a marker into a tree the user did not choose.
- **Every report root lies inside a Settings root, and the report is not the same list** (`ddc scan ~/Projects`, a documented use). The notice only: "This report is of ~/Projects only. Scan Now covers your home folder and the folders in Settings." Clean works there today (`clean_items.rs` drops the report, takes a full walk, and the plan accepts the items), so Clean and Protect stay on.

**The CLI.** `ddc scan <root>…` with roots other than the settings', and with neither `--data-dir` nor `DDC_DATA_DIR`, prints one line on stderr: "The app will show this report until its next scan." Nothing else changes in the CLI: it keeps writing `last-scan.json`, because `ddc report` and `ddc clean` read it.

## Decomposition

- [x] DDCKit: the roots helper (realpath, existence, sort, dedup, nested) and the comparison with `scanRoots` and `home`; the outside/inside/same result.
- [x] DDCKit: the wording for both cases, one and several roots, through `Wording`.
- [x] App: the notice in the Review window (above the groups) and in the popover (in place of "found in …"), with Scan Now; Clean and Protect disabled in the outside case.
- [x] CLI: the stderr line in `scan` when the roots differ from the settings' and the data folder is the default one.

## Implementation Plan

From the planner's placement (2026-10-29):

1. CLI: `Env::data_dir_is_default` (`crates/ddc-core/src/env.rs`, set in `from_process`, false in `with_paths`); in `cmd/scan.rs` the settings are loaded once, and after `write_last_scan` a scan given paths, into the default data folder, whose report fails `recheck::same_roots` against the settings' own roots prints `ddc: The app will show this report until its next scan.` Tests in `crates/ddc-cli/tests/scan_cli.rs`.
2. DDCKit: `ReportScope.swift`, the `same`/`inside`/`outside` verdict: `realpath(3)` through Darwin with the path kept when it does not resolve; the Settings roots or the home; existing ones canonical, missing ones matched as written and left out; nested dropped with `FolderList.isInside`; compared as sets (Swift and Rust sort paths differently); another home is `outside`.
3. `Wording.reportScope` beside `Wording.scope`: one root through `tilde`, several counted.
4. `PopoverModel` and `ReviewModel` take `home:` (default the user's, as `SettingsModel`); the verdict is cached when the report or the settings change and on `refresh()` (a disk plugged in), never computed on a redraw. Gates: `canCleanSelected` and `cleanSelected()`; `canClean`, `openSheet` (also the popover's `confirm: true`), `canProtect`. Test rigs default `home:` to `/Users/example`, the fixtures' home.
5. Views: `Banner` gains an optional button; the Review window shows the notice above the groups with Scan Now (hidden while paused, off while a scan runs), not in the Whole disk view; the popover shows it in place of "found in …" and in "Nothing found", with Scan Now; Clean Selected… follows `canCleanSelected`.
6. Tests: `ReportScopeTests.swift` (verdicts, wording, both models' gates); the AC5 manual check stays the owner's.

## Acceptance Criteria

- [x] Test (DDCKit): a report of `/private` with Settings roots `[home]` is "outside"; a report of a folder under the home is "inside"; Settings has an extra root that is unplugged and the report lacks it: "same"; a report whose extra root is a Settings root unplugged now is "same"; a root given through a symlink and a `/private`-prefixed path compare as the CLI compares them. — evidence: `ReportScopeTests.swift` `canonical`, `verdicts` (commit dd938d2; `swift test` 280 passed)
- [x] Test (DDCKit): the wording for one root and for several roots, in both cases. — evidence: `ReportScopeTests.swift` `wording`, with the another-home sentence
- [x] Test (DDCKit): in the outside case the Review model offers no Clean and no Protect, and the popover's "Clean selected…" is off; in the inside case all stay as today; a report with equal roots and another home is "outside". — evidence: `ReportScopeTests.swift` `reviewGates`, `popoverGates`, `noticeGoesWithTheReport`, `unreadableSettings`
- [x] Test (CLI): `scan <tmp>` with a temp HOME and no `--data-dir` prints the stderr line; with `--data-dir`, with `DDC_DATA_DIR`, or with the settings' own roots, it does not. — evidence: `crates/ddc-cli/tests/scan_cli.rs` `scan_of_other_roots_into_the_app_data_folder_says_the_app_shows_it`, `scan_of_the_settings_roots_by_path_says_nothing` (`make gate-fast`, 474 passed)
- Manual checks (1) moved to [Owner checks: manual acceptance across stories](../../../ops/owner-checks-manual-acceptance-across-stories.md) on 2026-10-30, by the owner's decision.

## Final Summary

- What changed (commit dd938d2): `ReportScope` in DDCKit compares the report's roots and home with the app's (Settings roots or the home, through `realpath`, unplugged roots left out). The Review window shows a notice above the groups and the popover in place of "found in …", each with Scan Now. When the scope is outside, Clean, Protect and the popover's Clean selected… are off. `ddc scan <path>` into the default data folder prints `ddc: The app will show this report until its next scan.` when the roots are not the settings' roots.
- Why: a terminal `ddc scan /private` saved into the app's data folder, and the app showed it as its own, with Clean and Protect working on it.
- Tests: `swift test` 280 passed (7 new in `ReportScopeTests.swift`); `make gate-fast` 474 Rust tests passed (2 new in `scan_cli.rs`); unsigned Debug app build passed. The reviewer found AC1–4 met; its should-fix items were applied before the commit.
- Left open: AC5 (manual, owner). Nits: the Review window does not re-judge the scope when a disk is plugged in while it stays open; the Banner's `fixedSize` text applies to every banner; the popover's "Scanning …" line names the old report's roots until the scan ends.

## Technical Notes

- After review (2026-10-29): the popover's "Nothing worth cleaning" shows the notice too (a small terminal scan lands there); a report made under another home says so in its own sentence; with `settings.json` unreadable, another home or a root outside the home still counts as outside, so Protect writes no marker there; the Review window drops the notice when it reopens without a report. The Review window re-judges the roots when its rows are rebuilt; the popover also on each refresh (a disk plugged in).
- Open: Scanning… in the popover still names the old report's roots ("Scanning private…") until the scan ends.
- `realpath(3)` from Swift: `Darwin.realpath`, freeing the result. Test it against `/private/tmp` and `/tmp`.
- The tests use a temp HOME; no test scans `/private`.

## Dependencies

- Research note "Removal wherever a size is shown: coding agents, protected folders and the whole disk" (Finding 1)
- "Settings: folders, types, notifications" (DDC-002): the Settings roots. The code this story reads has landed; it does not wait for that story to close

## Attachments

-

---

*Last updated: 2026-10-30*

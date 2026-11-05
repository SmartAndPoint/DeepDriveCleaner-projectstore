---
type: story
id: "story-popover-and-review-what-grew-since-the-earlier-scan"
epic: "DDC-002"
title: "Popover and Review: what grew since the earlier scan"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-05
updated: 2026-11-05
external_refs: {}
tags: [app, growth, review, popover]
code_refs:
  - app/DeepDriveCleaner/AppDelegate.swift
  - app/DeepDriveCleaner/PopoverView.swift
  - app/DeepDriveCleaner/ReviewGrewView.swift
  - app/DeepDriveCleaner/ReviewView.swift
  - app/DeepDriveCleaner/ReviewWindowController.swift
  - app/Packages/DDCKit/Sources/DDCKit/CLIRunner.swift
  - app/Packages/DDCKit/Sources/DDCKit/ChangesReport.swift
  - app/Packages/DDCKit/Sources/DDCKit/Formatting.swift
  - app/Packages/DDCKit/Sources/DDCKit/GrewEntry.swift
  - app/Packages/DDCKit/Sources/DDCKit/PopoverModel.swift
  - app/Packages/DDCKit/Sources/DDCKit/ReviewModel.swift
  - app/Packages/DDCKit/Sources/DDCKit/ReviewRouting.swift
  - app/Packages/DDCKit/Sources/DDCKit/RunnerActivity.swift
  - app/Packages/DDCKit/Sources/DDCKit/ScanReport.swift
  - app/Packages/DDCKit/Sources/DDCKit/SpaceReport.swift
  - app/Packages/DDCKit/Tests/DDCKitTests/ChangesTests.swift
  - app/Packages/DDCKit/Tests/DDCKitTests/ContractTests.swift
  - app/Packages/DDCKit/Tests/DDCKitTests/GrewEntryTests.swift
  - app/Packages/DDCKit/Tests/DDCKitTests/Fixtures/changes-compared.json
  - app/Packages/DDCKit/Tests/DDCKitTests/Fixtures/changes-not-compared.json
  - app/Packages/DDCKit/Tests/DDCKitTests/PopoverSupport.swift
  - app/Packages/DDCKit/Tests/DDCKitTests/RunnerActivityTests.swift
  - crates/ddc-cli/src/cmd/changes.rs
  - crates/ddc-cli/tests/cases.json
  - crates/ddc-core/src/changes/compare.rs
  - crates/ddc-core/src/changes/mod.rs
  - crates/ddc-core/tests/changes.rs
  - docs/changes.schema.json
specs: []
started_at: "2026-11-05T09:56:43.466Z"
closed_at: "2026-11-05T10:01:29.574Z"
plan_updated_at: "2026-11-05T09:56:43.466Z"
---

# Popover and Review: what grew since the earlier scan

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | done |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

S4 of the research [[research/what-grew-since-the-last-scan-a-ux-concept-for-the-macos-app|What grew since the last scan: a UX concept for the macOS app]]: the app says *where* used space went, with the numbers `ddc changes` already gives in a terminal (story "ddc changes: what grew between the last two scans", DDC-001). The owner picked it on 2026-11-05 and agreed to the plan: two pull requests, the code's group labels, no release until the owner asks (verify with builds and tests).

Today the popover's band (story "Popover: say when used space grew since the last scan") says only *that* space grew since the last scan, before a scan looks. After the scan the band goes, and nothing in the app names what grew between the two scans.

### The rule stays in Rust

`ddc changes --json` gains one top-level field, `grew: bool` (`#[serde(default)]`, always written): true when the result compared, `disk` is present and `disk.delta` is at least `space::grew_threshold` of the current volume's total (max(20 GiB, 5 %)), the same rule as `ddc space`'s `grew` (ADR "Menu bar app: Swift shell over the ddc CLI" §1: the app compares no thresholds). `CHANGES_SCHEMA_VERSION` stays 1, by the rule in `model.rs` ("New optional fields keep SCHEMA_VERSION"), as `Volume.read_at` and S1's fields did; the app reads it as required, since it ships its own CLI.

### The call

`CLIRunner` gains `changes`, queued like `space` (no lock, no lane). `PopoverModel.checkChanges()` runs it at the end of `loadReport()`, awaited, whatever loaded the report: the launch, a refresh that sees a new file (a terminal scan, a scheduled one), the app's own scan (so Scan Again in Review reloads with it), the scheduled scan in the running app, and `reloadReport()` after a clean (so the window's after-clean reload sees the new result and the entry stays). At most once per loaded report (`written_at`), not again after a failure for that report, and never while an update waits to install. It is decoded off the main actor.

The result is shown only while it belongs to the loaded report: `ScanReport` (DDCKit) decodes `written_at` (optional; a report before 0.31 has none, and then nothing shows), and a result whose `current.written_at` differs is not used. `not-compared` is a normal answer: no band, no entry.

The decoder (`ChangesReport.swift`) is strict as `SpaceReport` is: unknown keys are skipped, the enums it reads (`Status`, `Reason`, `ItemKind`, `HomeReason`, `NotReadKind`, `OriginHarness`) refuse a value they do not know, another `schema_version` is `otherSchema`. It reads `written_by` as a plain string, not an enum (a case `.clean` would collide with the contract test `cleanHasOneOwner`).

### The popover band (PR 1)

The band gains one state, "attributed":

```
↑ 198 GiB more in use since yesterday 10:04
  Build outputs +186 GiB · 14 new
  See What Grew
```

- **When**: the result's `grew` is true and it belongs to the loaded report; not during a scan; while that report stays loaded (the next scan replaces it). It never shows a shrink, and it lights no badge: the badge stays S5's signal's, and `runSpace`'s "the popover shows a band" check looks at S5's band only.
- **Precedence**: low disk (warning ink) wins; then S5's signal band (growth since the last scan, with Scan Now); then the attributed band, which has no Scan Now. One band at a time.
- **First line**: "↑ \<disk.delta\> more in use since \<when\>". \<when\> is the baseline's `written_at` said as "today 10:04", "yesterday 10:04" or a weekday and time (a new `Wording.since`, as `Wording.nextScan` says times); when the baseline was written by a clean (`baseline.written_by` is `clean` or `clean-items`), "than after your clean at 15:20" today, "than after your clean yesterday 10:04" before.
- **Second line**, chosen by where `disk.delta` went, in this order:
  1. `rest.delta` ≥ half of `disk.delta`: "Most of it is in folders ddc did not read or outside your home folder" (when the home did not compare, "outside the folders both scans read"), link "Where Did It Go?".
  2. the home compared and `home.elsewhere` ≥ half of `disk.delta`: "Most of it is elsewhere in your home folder, in folders ddc does not clean", link "See What Grew".
  3. otherwise the groups: the biggest group when its `delta` is at least 60 % of the positive group deltas' sum ("Build outputs +186 GiB · 14 new", or "· 3 grew" when none is new; "· mostly in Claude Code worktrees or data" ("· mostly in git worktrees" for hand-made ones) when one harness holds more than half of the group's `delta`), else "in N groups"; link "See What Grew".
  - When the positive group deltas sum to zero, rule 1's text is used, with "Where Did It Go?".
  - Whichever line is chosen: when the person's cleans since (`cleaned_since.deleted + trashed`) are at least a quarter of the positive group deltas' sum, it adds "· you cleaned 100 GiB since", so a group's +186 GiB under a first line of +96 GiB reads true; when `compared_roots` are narrower than the loaded report's roots, it adds "· compared in ~/Projects only".
- Group labels are the code's (`ddc rules --json`'s groups: "Build outputs", not the research's "Rust build folders").
- While scans are paused, the text stays; the popover already says "Scans paused".
- **The link** routes a `ReviewRequest` with a new `grew` section (`top` or `notRead`) and `keepsSelection: true`: an open window keeps the person's ticks and reloads (a new branch of `ReviewModel.open`, in PR 1); a closed one opens with the default selection. In PR 1 the window then shows what it shows today (no entry yet); PR 2 turns it to the entry.

### The Review entry (PR 2)

A sidebar row "What grew +191 GiB" above the class sections (its tooltip and VoiceOver label say "Grew since yesterday 10:04"). Clicking it, or the link, switches an Items-mode window to Groups.

- **Shown** when the result compared and either a changed row (`new`, `grew`, `moved`) matches a Review row, or `grew` is true (then the sections below carry it, with or without rows). Not on `not-compared`, a first scan, or a report without `written_at`.
- **The sidebar number**: `home.delta` when the home compared and it is positive; else the positive group deltas' sum, as "+188 GiB in findings"; no number when neither is positive.
- **Header**: "Grew since yesterday 10:04", and one line of the levels: "198 GiB more in use on the startup disk: 191 GiB in your home folder, 7 GiB in folders ddc did not read or outside your home folder." When the home does not compare: "Home total not compared: the two scans read different folders." (with `different-access`: "…: one scan had Full Disk Access and the other not."). With narrower `compared_roots`: "Compared in ~/Projects only: the earlier scan covered other folders."
- **Rows**: the same `ReviewItems` rows as the group entries, matched by `(path bytes, rule)` (as `ReviewModel.Identity` keys rows, not by `String`) to the result's `new`, `grew` and `moved` rows; rows with a `userEntry`, and rows locked as `typeOff` or `underMinimumSize`, are not matched (as `ReviewItems.rows(in:)` filters). They are grouped by the Review row's own `(riskClass, group)` (user data's per-folder names, not `changes`' single "Your data"), groups by their summed delta, falling. Same checkboxes, ticks, locks and "not selected" reasons; ticking here ticks the same item everywhere. A group header has a checkbox that ticks its shown rows (`toggleAll`, locked ones skipped) only where the group's own entry has Select all (`layout.selectAll` not nil: never user data, the Trash or session-only groups). The entry has **no "Clean N from …" button**: the footer's existing "Clean N items…" is its only way to the sheet (the research's mock-up drew one; left out so the entry adds no sheet caller).
- **A Change column** ("new", "+3.1 GiB", "moved", "≈" before an `approximate` one) in the place of the Changed column, sorted by Change, biggest first, by default in this entry, with its own sort and header (`GrewEntry.Sort`: Name, Size, Change), not the shared `ReviewSort` the group entries keep. The window's filter applies to the entry's rows too, and its Select all covers the rows it shows.
- **"Elsewhere in your home folder +3.0 GiB"**: one line, `home.elsewhere`, no checkbox: "ddc does not clean these." Folder sizes are S3.
- **"In folders ddc did not read, or outside your home folder +7.0 GiB"** (`rest.delta`): counts by kind of `rest.not_read`, each with its action: prompting areas with Scan Now, plus Open Privacy Settings only when the current scan ran without Full Disk Access (the report's own fact, as the Review window's background-skipped note reads it); skipped folders with Settings…; cloud folders as text ("not read by any scan"); outside the home folder with Whole Disk…. When `cleaned_since` is not zero: "Space you cleaned can read here until macOS frees it (local snapshots hold it)."
- **"Cleaned by you or gone (N)"**: folded by default; `cleaned`, `gone` and `shrank` rows as text, no checkboxes; "You cleaned 40 GiB since then; 12 GiB of it is in the Trash" from `cleaned_since`.
- The footer and the sheet are unchanged: the sheet stays the only thing that starts a clean, through the existing `openSheet` paths.

**What this story does not do**: folder sizes for growth that is no finding (S3), a growth notification (S6), Whole disk's Change column and a snapshots sentence from the whole-disk report (S7), "since yesterday / last week" (S8).

## Decomposition

PR 1, the field and the band:

- [x] Rust: `grew` in `ChangesReport` (`changes/mod.rs`; false in `not_compared`), computed in `compare.rs` from `disk()`'s current total with `space::grew_threshold`, in `i128` as `space::decide`; the schema regenerated; the `cmd/changes.rs` test helper; `cases.json`'s fresh-home case says `"grew": false`; ADR §6 item 15 gains a dated line
- [x] Rust test `the_apps_changes_fixtures_are_reports_the_cli_wrote` (as `tests/space.rs` makes the space fixtures): `changes-compared.json` and `changes-not-compared.json` in DDCKit's fixtures, from `scan-report-groups.json` and a baseline edited in memory, validated against the schema; run on the Windows host
- [x] DDCKit: `ChangesReport.swift` (strict, the fields above); `ScanReport.writtenAt`; `CLIRunner.Kind.changes`; `RunnerActivity` counts it with the scans; `PopoverModel.checkChanges()` and `attributedChanges`; `growthBand` = S5's band ?? the attributed one, `runSpace` checks S5's only; `GrowthBand.attributed` with the line rules and a `link`; `Wording.since`; `ReviewRequest.grew` (`GrewSection`); `PopoverModel.seeWhatGrew`; `ReviewModel.open`'s branch for a `grew` request (open window: reload, ticks kept)
- [x] `PopoverView`: the attributed band and its link, leaving what `ContractTests.spaceCallers` pins as it is
- [x] Tests: `ChangesTests.swift` (decoding both fixtures, strictness, the dropped result, every band state and precedence, the line rules, the link's request); `ContractTests.changesContracts` (version, enums against `$defs`, the fixtures validate, every key read is in its `$def`) and `changesCallers`; `RunnerActivityTests`; `FakeRunner` answers `.changes`

PR 2, the entry:

- [x] DDCKit: `GrewEntry.swift`, a value from `(ReviewItems, ChangesReport, sort)`: matching, groups, Select all where the layout has it, header, sidebar number, the three sections, its own `Sort` and `filtered(items:query:)`; `ReviewSnapshot.changes`; `ReviewModel` `grewSince` (reset wherever `wholeDisk` and the entry change), `grewSort`, `open` for a `grew` request
- [x] Views: `ReviewGrewView.swift` (the entry, its sidebar row, its header, a scroll to the not-read section), `ReviewView` (the row and the branch), `ItemRow`'s Change; the not-read actions passed from `AppDelegate` through `ReviewWindowController` as `wholeDisk` is
- [x] Tests: `GrewEntryTests.swift`; `ContractTests.grewEntryReachesNoSheet` (no `openSheet`, `cleanSelected`, `cleanConfirmedItems`, `confirm: true`, `SelectAllRow(` or `PickedRow(` in `ReviewGrewView.swift`; `openSheet(` callers unchanged)
- [x] Owner checks in the runbook "Owner checks: manual acceptance across stories"

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

From the planner's pass and two critic rounds (2026-11-05), no covering spec. PR 1 (#126): the Rust `grew` and the app's fixtures; `ChangesReport`, `ScanReport.writtenAt`, `CLIRunner.Kind.changes`; `PopoverModel.checkChanges` from `loadReport()` and the app's own scan; `GrowthBand.attributed` and the link; `ReviewModel.open` for a `grew` request. PR 2: `GrewEntry` (a value over `ReviewItems`), `ReviewSnapshot.changes`, the `ReviewModel` state, `ReviewGrewView.swift` and the sidebar row; tests at each step; `make ddckit-test` and a Debug `xcodebuild` before each push, the native legs for PR 1's Rust.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

PR 1:

- [x] Test (core): `grew` is true exactly when the result compared, `disk` is present and `disk.delta` ≥ `grew_threshold(total)` (at the threshold true, a byte under false, on a 1000 GiB and a 100 GiB volume); false when not compared, without `disk`, or on a shrink; the committed schema equals the generated one; the app's fixtures are what `compare` writes and validate against it — evidence: `grew_is_true_exactly_when_the_disk_grew_by_the_threshold`, `committed_changes_schema_is_current`, `the_apps_changes_fixtures_are_reports_the_cli_wrote` (crates/ddc-core/tests/changes.rs); Windows leg 1026/1026; Gate on #126 and on main for 8930772
- [x] Test (DDCKit): both fixtures decode; an unknown enum value and another `schema_version` are refused, an unknown key is skipped; a result whose `current.written_at` is not the loaded report's, or a loaded report without `written_at`, shows nothing — evidence: ChangesTests `decodes`, `strict`, `otherReport`
- [x] Test (DDCKit): the attributed band shows only with `grew` and its report, not during a scan; low disk and then S5's band win over it; it has no Scan Now and sets no badge; the second line follows the order: mostly not read or outside (with "Where Did It Go?"), mostly elsewhere in the home, one group at ≥ 60 % (new count, or grew count, and the harness), else "in N groups"; the cleans clause at a quarter; the compared-roots clause; "after your clean at" for a clean's baseline; never a shrink; paused keeps the text — evidence: ChangesTests `model`, `ownScan` (none during a scan), `precedence`, `lowDiskAndPaused`, `badge`, `firstLine` ("after your clean at" today, a day before), `noBand`, `mostlyOutside`, `mostlyElsewhere` (with no group grown), `groups`, `clauses`
- [x] Test (DDCKit): the link routes a `grew` request with `keepsSelection`; with the window open, `ReviewModel.open` keeps the person's ticks — evidence: ChangesTests `link`, `reviewKeepsTicks`
- [x] Contract tests: the decoder's version and enums equal the schema's; only `PopoverModel` runs `changes` — evidence: ContractTests `changesContracts` (version, six enums, every key read, the fixtures, `WrittenBy`), `changesCallers` (one run, from `loadReport` and `runScan`)
- [x] The Windows and Linux legs, `make ddckit-test` and the Gate pass; a Debug `xcodebuild` of the app target compiles (the Gate builds none) — evidence: Windows leg 1026/1026 with native-skips; Linux gate-fast in the WSL hooks and the PR's check-linux; `make ddckit-test` 513 passed; Debug `xcodebuild` compiled; Gate green on #126 and main 8930772

PR 2:

- [x] Test (DDCKit): the entry shows on a matched `new`, `grew` or `moved` row or on `grew` alone (then with no rows and its sections), not on `not-compared` or a first scan; rows are matched by path bytes and rule, never a `userEntry`, `typeOff` or `underMinimumSize` row; grouped by the Review row's group (Downloads in its own); sorted by Change with its own sort; a tick in the entry is the same tick in the group's entry; the header checkbox only where the group's layout has Select all; the sections have no item ids; the sidebar number is `home.delta`, else the positive groups' sum, else none — evidence: GrewEntryTests `rows`, `shown`, `matching` (another rule, another spelling, `moved` with "≈"), `filters` (typeOff, underMinimumSize, a user-data entry), `groupsAndSelectAll`, `sort`, `filter`, `amount`, `notReadAndCleans`, `window` (the same ids as the group's rows)
- [x] Test (DDCKit): a `grew` request with the window open keeps the person's ticks and shows the entry; Scan Again rebuilds it from the new result; a clean from the entry leaves the window on the entry, rebuilt; leaving it for a group or Whole disk clears it — evidence: GrewEntryTests `window`, `rebuilds` (Scan Again, the reload after a clean with the cleaned row gone, a result that compares nothing)
- [x] Structure test: `ReviewGrewView.swift` reaches no sheet, and the `openSheet(` callers are the same files as before — evidence: ContractTests `grewEntryReachesNoSheet`
- [x] `make ddckit-test` and the Gate pass; a Debug `xcodebuild` of the app target compiles — evidence: `make ddckit-test` 526 passed (and the pre-push hook's run); Debug `xcodebuild` compiled; #127's checks green (app, scope, the check legs) and main's Gate on 454cb0f
- [ ] Owner check (runbook): after a day of builds and a Scan Now, the popover names the group that grew and See What Grew opens the entry with those rows, ticked as in their group; Elsewhere and the not-read section match `ddc changes`

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->


**What changed.** `ddc changes --json` gains `grew`, `ddc space`'s rule on the two reports' in-use space (schema stays 1). The app runs `ddc changes` after each report load and after its own scan, once per report (`written_at`, now decoded in `ScanReport`). The popover's band, when it grew and S5's signal and low disk have nothing to say, tells how much more is in use since the earlier scan and where: mostly in folders ddc did not read or outside the home folder, mostly elsewhere in the home folder, the group that grew most (with its new count and coding agent), or "in N groups"; with "you cleaned … since" and "compared in … only" when they apply. Its link opens the Review window's new "What grew" entry (PR 1 #126, 8930772): the window's own rows that are new, grew or moved, grouped by their Review group, with a Change column and their own sort, ticks shared with the groups and the footer still the only way to the sheet; then elsewhere in the home folder, what the scan did not read with its actions, and what the person cleaned or what went (PR 2 #127, 454cb0f).

**Why.** S4 of the research "What grew since the last scan: a UX concept for the macOS app": after S2 the terminal could say where the space went, the app could not.

**Tests.** Rust: the `grew` boundary and the fixture test that writes the app's `changes` fixtures from `compare`. DDCKit: `ChangesTests` (decoding, strictness, every band state and line rule, the model's calls, the badge, the link), `GrewEntryTests` (matching, filters, groups, Select all, sort, filter, sections, the window flow, rebuilds), `ContractTests` (`changesContracts`, `changesCallers`, `grewEntryReachesNoSheet`), `RunnerActivityTests`. DDCKit 526 passed; Windows leg 1026/1026; Linux gate-fast; Debug `xcodebuild` of the app target before each push; Gate green on both PRs and on main.

**Risks and follow-ups.** Not seen on a real Mac yet (no release until the owner asks): the owner checks are in the runbook, including whether the band should go once the popover was opened. Open Privacy Settings follows the report's own Full Disk Access, not the app's now. The scroll to the not-read section uses an anchor on a `List` section, which may not land on every macOS. Folder sizes for growth elsewhere are S3; the growth notification S6; Whole disk's Change column S7; "since yesterday / last week" S8. The Gate builds no app target, so the Debug build stays a manual step before a push.

## Technical Notes

- The research's words ("Rust build folders", "Folders that are no finding") are the concept's; this story uses the code's group labels and one Elsewhere line until S3.
- Two Scan Nows in a row compare a minutes-old baseline: `grew` is false, so no band (the research's "No change since 10 min ago" is left out on purpose), and the entry shows whatever matched growth there is.
- The attributed band stays while its report is loaded, which can be days; whether it should go once the popover was opened is the owner check's to say.
- `changes` parses two reports and the journal on the CLI queue, well under a second on a large home; a Scan Now waits for it.
- After a `clean --items` edit the loaded report's `written_at` changes; `reloadReport()` awaits the new result, so the window's after-clean reload rebuilds the entry from it. A report a terminal wrote shows in the window at its next open, reload or Scan Again.
- Built in PR 2: the entry's order is its own `GrewEntry.Sort` (Name, Size, Change) with its own header, not a `.change` column added to the shared `ReviewSort` and `SortHeader`; the rows show their Change in the Changed column's place. The not-read section's Settings… opens Settings → Folders.
- `ByteSize.human` writes "198.0 GiB"; the band uses whatever the popover uses today.

## Dependencies

- Story "ddc changes: what grew between the last two scans" (DDC-001): the command and its schema.
- Story "Popover: say when used space grew since the last scan" (DDC-002): the band, `GrowthBand`, `checkSpace` and the badge.
- ADR "Menu bar app: Swift shell over the ddc CLI" §1 (rules in Rust), §6 items 13-15.

## Attachments

-

---

*Last updated: 2026-11-05*

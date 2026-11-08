---
type: story
id: "story-popover-after-an-update-say-why-a-new-scan-is-needed"
epic: "DDC-002"
title: "Popover: after an update, say why a new scan is needed"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-02
updated: 2026-11-08
external_refs: {}
tags: [app, popover, updates]
code_refs:
  - app/Packages/DDCKit/Sources/DDCKit/OtherVersion.swift
  - app/Packages/DDCKit/Sources/DDCKit/ScanReport.swift
  - app/Packages/DDCKit/Sources/DDCKit/DiskReport.swift
  - app/Packages/DDCKit/Sources/DDCKit/PopoverModel.swift
  - app/Packages/DDCKit/Sources/DDCKit/DiskModel.swift
  - app/Packages/DDCKit/Sources/DDCKit/BrewPackagesModel.swift
  - app/Packages/DDCKit/Sources/DDCKit/AgentsModel.swift
  - app/DeepDriveCleaner/PopoverView.swift
  - app/DeepDriveCleaner/DiskView.swift
  - app/DeepDriveCleaner/AppDelegate.swift
specs: []
started_at: "2026-11-08T11:35:14.275Z"
closed_at: "2026-11-08T12:02:45.059Z"
plan_updated_at: "2026-11-08T11:35:14.275Z"
---

# Popover: after an update, say why a new scan is needed

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner, 2026-11-02, on a screenshot: after an update the popover often says "No scan yet" and "A scan reads your home folder and changes nothing", as on a first launch, although the Mac was scanned many times. Since 0.14.10 a `last-scan.json` written by an older schema counts as no report (`PopoverModel`, the "older schema" branch), which removed the error but kept the first-launch wording. A report schema changes often (scan 4 → 5 is planned with the Linux trash work), so this happens after most updates.

The popover should say what it knows, not guess. An older-schema report has two sources: an update of the app, or a `ddc` of another version (a Homebrew CLI, a legacy LaunchAgent) writing into the same data folder (`cmd/scan.rs`: "The app will show this report until its next scan"). So:

- **The text names the report's maker:** "Your last scan was made by ddc 0.17.4. This version reads reports differently, so it needs a new scan." It says "after the update" only when the report's `ddc_version` is older than the app's own version; when it is newer ("made by ddc 0.19.0"), it says so and suggests updating the app, as the unreadable-newer case does today.
- **Scan Now is the main action;** the line below gives the next scheduled scan as `ScanDue` computes it (which may be days away), not a fixed "within the hour".
- **The header subtitle** reads "Last scan by ddc 0.17.4 · scan again", not "No scan yet". The menu bar icon shows no warning, as since 0.14.10.
- **When the report's version cannot be read** (no `ddc_version` in the older file), the text omits it: "Your last scan was made by an earlier version of ddc".
- The Whole Disk view's "Scan Whole Disk Again" (0.14.10) uses the same wording family, from `last-disk.json`'s header.

*Superseded 2026-11-04 by the owner: the story "Scan after an update when the saved report cannot be read" makes the scheduled scan due at once after an update, when scheduled scans are on and not paused; this story keeps the wording for the cases left (scheduled scans off or paused, a report of a newer `ddc`).* **No automatic scan.** Considered and dropped: the menu bar ADR (§5, §6) knows two kinds of scan, a person's and the scheduled one, and a third, self-started kind would scan at full speed at relaunch and could raise the privacy prompts the background path avoids. The person starts it with Scan Now.

## Decomposition

- [x] DDCKit: `VersionHeader` also decodes `ddc_version` (optional); `ScanReport.ReadError.otherSchema` carries it
- [x] `PopoverModel`: a state for an older- or newer-schema report with the maker's version, `Content.otherVersion`, the header subtitle
- [x] `PopoverView`: the block with Scan Now and the next scheduled scan from `ScanDue`
- [x] `DiskModel`/`DiskView`: the same wording family from `last-disk.json`'s header
- [x] DDCKit tests

## Implementation Plan

Planner run 2026-11-08 against `origin/main` c907b35. One Swift-only PR (DDCKit and the app); the push runs `make ddckit-test`.

What exists: `appVersion` already reaches `PopoverModel` (AppDelegate, `CFBundleShortVersionString`, equal to the CLI's version); an older-schema report becomes `report = .none` (so `.firstScan`, "No scan yet"), a newer one `.unreadable` with the `.unreadableReport` attention; `VersionHeader` reads only `schema_version`; the Whole Disk view's "made by an older version" text is hard-coded in `DiskView.swift`. AC 3 (no file → `firstScan`) holds today and is kept; AC 7 is covered by `ScheduledScanTests.updateScanNotNeeded` and gets one popover-side test.

1. **The header and the error.** `VersionHeader` gains `ddcVersion: String?`, decoded softly (a missing or mistyped `ddc_version` is nil, never an error: about twelve readers share the header). `ScanReport.ReadError.otherSchema(Int, ddcVersion: String? = nil)`; `ScanReport.read` and `DiskReport.read` fill it. Every `case .otherSchema(let version)` pattern (PopoverModel twice, DiskModel twice, BrewPackagesModel, AgentsModel) is rewritten by hand: a one-name pattern on a two-value case binds the tuple and would print "(schema (2, nil))".
2. **`OtherVersion`** (new `OtherVersion.swift`): the report's schema and the app's, the maker's `ddcVersion` and the app's version. Readability comes from the schema (older or newer than the app's); the words come from the versions, compared with `compare(_:options: .numeric)` as `UpdateModel` does. Pure functions in `Wording`:
   - maker older than the app: title "Scan again after the update"; text "Your last scan was made by ddc 0.17.4. After the update, this version reads reports differently, so it needs a new scan.";
   - maker newer than the app: title "Update the app, or scan again"; text "Your last scan was made by ddc 0.19.0, a newer version than this app. Update DeepDriveCleaner to read it, or scan again.";
   - same version, or no app version (a development build): the number, and the direction from the schema: "… made by ddc 0.34.0 in an older report format, so this version needs a new scan." (title "A new scan is needed") or "… in a newer report format. Update DeepDriveCleaner to read it, or scan again." (title "Update the app, or scan again");
   - no `ddc_version` in the file: "Your last scan was made by an earlier version of ddc, which this version reads differently, so it needs a new scan." or, for a newer schema, "… by a newer version of ddc. Update DeepDriveCleaner to read it, or scan again." No number, no "after the update". (Every report has had `ddc_version` since the first commit, so this case is rare.)
   - the header subtitle: "Last scan by ddc 0.17.4 · scan again" ("by an earlier ddc" without a number), "… · update the app" for a newer one;
   - the Whole Disk view, the same three directions with the verb "scan": "The last whole-disk report was made by ddc 0.17.4. After the update, this version reads it differently, so it needs a new scan. A new scan takes a few minutes."
3. **`PopoverModel`.** `ReportState` is unchanged (`.none` and `.unreadable` keep the growth band and the attention list as they are). Beside it, `otherVersion: OtherVersion?`, set by both `otherSchema` branches of `apply`, cleared by every other branch and by "no file". `Content.otherVersion(OtherVersion)` wins over `.firstScan` and `.unreadableReport`; `.scanning` still wins over it. The subtitle uses the line above. The next scheduled scan is the footer's line, `nextScanText`, which `ScanDue` computes ("Next scan within the hour" while the update scan is due, "Scheduled scans off" when off, "Scans paused · Resume" when paused): the block adds no second copy of it.
4. **`PopoverView`.** A block like `FirstScanView`'s: the title, the text, Scan Now `.borderedProminent` (off while `!canScan`, which also covers a waiting update). Without Full Disk Access it keeps `FirstScanView`'s form for that case: Open Privacy Settings prominent, Scan Anyway as the link, and the version text above them, so a scan without the grant never becomes the main action.
5. **`DiskModel`/`DiskView`/`AppDelegate`.** `ReportState.outdated(OtherVersion)`; `DiskModel` gets `appVersion` (AppDelegate passes it); the outdated title and text move from `DiskView` to DDCKit, so a unit test reads them.
6. **Tests.** `makeModel` gains `appVersion`; reports through `Fixture.edited` (schema − 1 with `ddc_version` "0.17.4", with it removed, "0.19.0"); `DiskModelTests` with an older `disk-report`. Updated: PopoverModelTests `olderSchema` and `newerSchema`, ScheduledScanTests `updateScanStaysDue`, ScanReportTests and DiskReportTests (`otherSchema` with the maker), DiskModelTests `.outdated`. New: one test per AC; a mistyped `ddc_version` (a number) is read as none, never an error; without Full Disk Access the block keeps Open Privacy Settings as its main action; AC 7 checks that `start()` and `refresh()` with an older report run no scan.
7. **Builds.** The Gate's `app` job runs `make app-test`, not a build of the app target, and this PR changes three files of it: a Debug `xcodebuild` of the app target runs before the push, as in "Scan after an update when the saved report cannot be read". At close, the runbook "Owner checks: manual acceptance across stories" gets the owner check.

Decisions recorded here: the icon keeps its warning for a newer-schema report (the app cannot read it); only the older-schema case shows none, as since 0.14.10. Equal versions name the number and take the direction from the schema. Without Full Disk Access the grant stays the main action (critic, 2026-11-08).

Code refs, proposed: `app/Packages/DDCKit/Sources/DDCKit/{ScanReport,DiskReport,OtherVersion,PopoverModel,DiskModel,BrewPackagesModel,AgentsModel,Formatting}.swift`, `app/DeepDriveCleaner/{PopoverView,DiskView,AppDelegate}.swift`, and the tests.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [x] Test (DDCKit): a `last-scan.json` of an older schema whose `ddc_version` is older than the app's gives the other-version content, not `firstScan`; its text names that version and says "after the update"; the header subtitle is not "No scan yet"; the icon's attention list is empty; Scan Now is offered; the next-scan line comes from `ScanDue` — evidence: `PopoverModelTests.afterTheUpdate` (version, "After the update", title, subtitle "Last scan by ddc 0.17.4 · scan again", empty attention, Scan Now); `ScheduledScanTests.updateScanStaysDue` (the footer's line from `ScanDue`: "Next scan within the hour") (#142)
- [x] Test (DDCKit): an older-schema report whose `ddc_version` is newer than the app's (a newer Homebrew CLI) does not say "after the update" and suggests updating the app — evidence: `PopoverModelTests.newerMaker`; `PopoverModelTests.newerSchema` for a newer schema, which keeps the icon's warning (#142)
- [x] Test (DDCKit): no `last-scan.json` at all still gives `firstScan` with today's wording — evidence: `PopoverModelTests.noFileIsFirstScan`, and the first-launch test that has no file (#142)
- [x] Test (DDCKit): when the older report's `ddc_version` can be read, the text names it ("made by 0.17.4"); when it cannot, the text omits it — evidence: `PopoverModelTests.afterTheUpdate`, `noMakerVersion`; `ScanReportTests.makerReadSoftly` (missing, a number, empty: none, never an error) (#142)
- [x] Test (DDCKit): the Whole Disk view's outdated text names the update in the same words — evidence: `DiskModelTests.outdatedNamesTheUpdate` (`Wording.otherVersionDiskText`, its title, the popover's sentence) (#142)
- [x] `make ddckit-test` passes, and a Debug build of the app target compiles (the Gate's `app` job runs `make app-test`, not that build) — evidence: `make ddckit-test` 585 tests on the Mac and in the pre-push; a Debug `xcodebuild` of the app target; PR #142's CI green
- [x] The app starts no scan of its own when scheduled scans are off or paused, or the report is of a newer schema (the story "Scan after an update when the saved report cannot be read" covers the rest) — evidence: `PopoverModelTests.noScanOfItsOwn`; `ScheduledScanTests.updateScanNotNeeded` (off, paused, newer) (#142)
- [ ] Owner check (runbook "Owner checks: manual acceptance across stories"): after the next update that changes a report schema, the popover names the version that made the last scan and asks for a new one

## Final Summary

**What changed.** PR #142. The report header's `ddc_version` is read softly (missing, empty or mistyped is none), and `ScanReport.ReadError.otherSchema` carries it from `ScanReport.read` and `DiskReport.read`. `OtherVersion` compares the maker's version with the app's and keeps the schema's direction; `Wording` gives its title, text and subtitle. The popover shows them in place of "No scan yet" (`Content.otherVersion`, beside an unchanged `ReportState`), with Scan Now, or Open Privacy Settings and a note when Full Disk Access is missing. The Whole Disk view's outdated report uses the same words, now from DDCKit.

**Why.** The owner, 2026-11-02: after most updates the popover said "No scan yet", as on a first launch.

**Tests.** `PopoverModelTests` (after the update, a newer maker, no version, a newer schema, no file, clearing, no scan of its own), `ScanReportTests.makerReadSoftly`, `DiskModelTests.outdatedNamesTheUpdate`, `ScheduledScanTests.updateScanStaysDue`, `ContractTests.otherVersionKeepsTheGrant`.

**Where it differs from the plan.** The words live in `OtherVersion.swift`, not `Formatting.swift`. The block has no next-scan line of its own: the footer's is the due function's (critic). The Whole Disk view's title follows the words too.

**Risks and follow-ups.** An older schema made by a newer ddc (a branch build) advises updating the app, which would not help; left for the owner to decide. The owner check below is open.

## Technical Notes

- `ScanReport.read` throws `otherSchema(version)` before decoding; reading `ddc_version` from an older file needs a small header decode (schema_version, ddc_version), as `VersionHeader` does for the schema version.

## Dependencies

- "Scan after an update when the saved report cannot be read" (DDC-002): it narrows this story to the cases left.
- Follows the 0.14.10 fix "a report from before an update asks for a new scan, not an error".

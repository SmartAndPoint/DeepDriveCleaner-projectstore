---
type: story
id: "story-full-disk-access-check-on-macos-27"
epic: "DDC-002"
title: "Full Disk Access check on macOS 27"
status: done
priority: p0
assignee: "Evgenii Konev"
created: 2026-11-05
updated: 2026-11-05
external_refs: {}
tags: [app, cli, permissions, macos, bug]
code_refs:
  - app/DeepDriveCleaner/AppDelegate.swift
  - app/DeepDriveCleaner/DebugMenu.swift
  - app/Packages/DDCKit/Sources/DDCKit/FullDiskAccessProbe.swift
  - app/Packages/DDCKit/Sources/DDCKit/ScanReport.swift
  - app/Packages/DDCKit/Tests/DDCKitTests/ContractTests.swift
  - app/Packages/DDCKit/Tests/DDCKitTests/FullDiskAccessProbeTests.swift
  - crates/ddc-core/src/report/mod.rs
  - crates/ddc-core/tests/report.rs
  - crates/ddc-cli/tests/background_cli.rs
  - .config/native-skips/windows.txt
specs: []
started_at: "2026-11-05T05:58:03.953Z"
closed_at: "2026-11-05T06:52:17.192Z"
plan_updated_at: "2026-11-05T05:58:03.953Z"
---

# Full Disk Access check on macOS 27

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | done |
| **Priority** | p0 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner, 2026-11-05: after updating the Mac to macOS 27, the popover says "Full Disk Access Needed", while System Settings → Privacy & Security → Full Disk Access shows DeepDriveCleaner switched on.

Both checks read one file. The app's `FullDiskAccessProbe` (DDCKit, live, for the popover and Settings) and the CLI's `report::full_disk_access` (ddc-core, for the report's `full_disk_access` and for which folders `ddc scan --background` skips) open `~/Library/Application Support/com.apple.TCC/TCC.db`: it opens only with Full Disk Access, and the open fails with `EPERM` without it. On macOS 27 that open fails whatever the grant, and the reports disagree on how: several projects find no such file (`ENOENT`; [cleardisk #40](https://github.com/bysiber/cleardisk/issues/40), [macos-apps-mcp #261](https://github.com/elfensky/macos-apps-mcp/issues/261)), a thread on Apple's developer forums finds it refused with the grant on (`EPERM`; [841091](https://developer.apple.com/forums/thread/841091)). Which one the owner's Mac gives we have not seen: reading `~/Library` there is not ours to do. The check below works for both.

What it breaks on macOS 27, with the grant on:
- The popover and Settings → Permissions say Full Disk Access is off, and the icon's attention state counts it.
- `ddc scan --background` (every scheduled scan) skips the folders macOS asks before reading (`PROMPTING_AREAS`: Desktop, Documents, Downloads and the rest), so the findings there are missing and their space shows as "elsewhere".
- Every report says `full_disk_access: false`, which `ddc changes` reads (`HomeReason::DifferentAccess`).

What the check should do: try more than one thing that only Full Disk Access can read, in a fixed order. Any one read means granted; an error of a step, whatever its errno (`ENOENT`, `EPERM`, `EACCES`, `ENOTDIR`), only sends the check to the next step; when none can be read, not granted.

1. The per-user `TCC.db`, opened as today.
2. `~/Library/Safari`, listed: open the folder and read one entry (an open alone may pass where the listing is refused); an empty folder that opens counts as read.

**Older macOS keeps today's answer.** On macOS 26, 25 and older the per-user `TCC.db` exists. With the grant it opens at step 1, as today. Without it step 1 fails, and so does step 2: `~/Library/Safari` has needed Full Disk Access since macOS 10.14, so the answer is "not granted", as today (the owner, 2026-11-05: "работа на 26 и может даже на 25 макос не сломалась"). A "final at `EPERM`" rule would protect nothing more there, and would miss the forum's macOS 27 case.

Both sides take the same list, in the same order. The CLI builds it from `Env::home()`, the app from the home folder in the app target only, so tests give temporary paths and no test opens the real `TCC.db` or `~/Library/Safari`.

The system database (`/Library/Application Support/com.apple.TCC/TCC.db`, which macos-apps-mcp probes instead and reports readable on 27.0.1) is left out of this story. It lies outside the home folder, so its path would come from `Env::system_root()`, and `ddc scan` and `ddc scan --background` have no `--system-root`: the tests that expect "not granted" in a temporary home (`tcc_areas_are_never_opened`, `tests/report.rs`'s `unreadable_folders_are_reported`) would read the real one, and a test machine whose runner has the grant would flip them. Adding it means that flag and those tests; it waits for evidence that `~/Library/Safari` does not answer on macOS 27 (the owner check). If a Mac has neither a readable per-user file nor `~/Library/Safari` (Safari never opened on that account), the answer stays today's: no regression, only no fix there.

## Decomposition

- [x] ddc-core `report/mod.rs`: `FULL_DISK_ACCESS_PROBES` (home-relative path and kind, in order) and `granted_by` over a given list; `full_disk_access_with` builds the list from `Env::home()`; directory reads through `stats::read_dir` (structure test `the_core_reads_directories_only_through_the_counters`); no `std::os::unix` in `src` (structure test `no_file_outside_the_platform_folders_names_an_os`), so an unreadable probe in the unit tests is a file where its folder should be, not mode 000
- [x] ddc-cli `tests/background_cli.rs`: a background scan with only a listable `Library/Safari` scans the prompting areas; its name in `.config/native-skips/windows.txt` (gated by `common::unsupported` only, like `with_full_disk_access_prompting_areas_are_scanned`; on Linux it passes because no capability withholds anything); the comment in `tests/report.rs`
- [x] DDCKit `FullDiskAccessProbe`: `homeRelative` (the same list), `locations(home:)`, `init(locations:)` (`init(path:)` stays, a one-file list, for the popover tests in `PopoverSupport.swift`, `RealCLISettingsTests.swift` and `PopoverModelTests.swift`), `steps()` (each step's errno, up to the first read) and `check()` (`.missing` carries the last step's errno); `FullDiskAccessProbeTests.swift`; a `ContractTests` case that compares `homeRelative` with `FULL_DISK_ACCESS_PROBES`; the comment in `ScanReport.swift`
- [x] App: `AppDelegate.fullDiskAccessProbe` over the real home replaces `tccDatabase`; the Debug menu's "Probe Full Disk Access" logs one line per step
- [x] Owner check in the runbook "Owner checks: manual acceptance across stories"

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

From the planner's pass (2026-11-05), no covering spec:

1. ddc-core `report/mod.rs`: `ProbeKind`, `FULL_DISK_ACCESS_PROBES`, `granted_by` (a file opened, a folder listed through `stats::read_dir`), `full_disk_access_with` over `Env::home()`; unit tests over temporary homes. Callers (`scan_with`, `cmd/background.rs`) unchanged.
2. ddc-cli `tests/background_cli.rs`: the Safari-only background scan; its name in `.config/native-skips/windows.txt`; the comment in `tests/report.rs`.
3. Native runs: the Windows leg and Linux `gate-fast` on the build host, whole runs.
4. DDCKit `FullDiskAccessProbe` (`Kind`, `Location`, `Step`, `homeRelative`, `locations(home:)`, `init(locations:)`, `steps()`, `check()`), `FullDiskAccessProbeTests.swift`, the `ContractTests` case, the `ScanReport.swift` comment; `make ddckit-test`.
5. App: `AppDelegate.fullDiskAccessProbe` and the Debug menu's per-step log, built by the Gate's `app` job.
6. The owner check in the runbook; `projectstore:reviewer`.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [x] Test (ddc-core, temporary paths, every platform): a readable `TCC.db` is granted; with no `TCC.db`, a listable Safari folder is granted, empty or not; a refused `TCC.db` with a listable Safari folder is granted (the forum's macOS 27 case); none, a refused `TCC.db` alone, and both refused are not granted (no grant, any macOS); without the `full-disk-access` capability it is granted as today — evidence: `a_readable_tcc_database_grants_full_disk_access`, `without_a_tcc_database_a_listable_safari_folder_grants_full_disk_access`, `a_refused_tcc_database_and_a_listable_safari_folder_grant_full_disk_access`, `when_no_probe_can_be_read_full_disk_access_is_not_granted`, `full_disk_access_reads_the_probes_in_the_home`, `full_disk_access_is_true_without_the_capability` (crates/ddc-core/src/report/mod.rs); Windows leg 1014/1014, Linux gate-fast, main's Gate on 5b001c1
- [x] Test (ddc-cli): `ddc scan --background` with no `TCC.db` and a listable `Library/Safari` in the temporary home scans the prompting areas and says `full_disk_access: true`, as `with_full_disk_access_prompting_areas_are_scanned` does with a `TCC.db` — evidence: `without_a_tcc_database_a_listable_safari_folder_scans_the_prompting_areas` (crates/ddc-cli/tests/background_cli.rs), run on macOS by main's Gate on 5b001c1; listed in .config/native-skips/windows.txt
- [x] Test (DDCKit): `FullDiskAccessProbe` gives the same answers, with refusals by mode 000 (`EACCES`, which takes the same path as `EPERM`), and stops at the first read (a readable `TCC.db` leaves the Safari folder untried); the contract test holds both lists equal — evidence: FullDiskAccessProbeTests.swift (`readableTCC`, `safariAnswers`, `neither`, `refusedTCCThenSafari`, `refusedTCCAlone`, `bothRefused`, `unlistableSafari`, `empty`, `locations`); ContractTests `fullDiskAccessProbes`; `make ddckit-test` 493 passed
- [x] Every test that expects "not granted" in a temporary home is unchanged and passes: `tests/report.rs`'s `unreadable_folders_are_reported` (`full_disk_access == !tcc`) and `background_cli.rs`'s `tcc_areas_are_never_opened` (a temporary home has neither path); the goldens hold `full_disk_access` as volatile — evidence: `unreadable_folders_are_reported` (crates/ddc-core/tests/report.rs, comment only changed) and `tcc_areas_are_never_opened` (unchanged) pass in main's Gate on 5b001c1; `full_disk_access` in VOLATILE (crates/ddc-cli/tests/golden_cli.rs)
- [x] The Gate passes on all legs and `make ddckit-test` passes — evidence: PR #123's Gate green on every leg, main's Gate on 5b001c1 success; `make ddckit-test` 493 passed; a Debug `xcodebuild` of the app target compiled (the Gate builds no app target)
- [ ] Owner check (runbook "Owner checks: manual acceptance across stories"): on macOS 27 with the grant on, the popover shows no "Full Disk Access Needed" and the report of the app's next scan (not `ddc scan` in Terminal, whose access is Terminal's) has `full_disk_access: true`; with the grant switched off, it says "Full Disk Access Needed" again; on a Mac with macOS 26 or older, if one is at hand, nothing changes. If the grant on still says "Needed", the owner runs `ls -la ~/Library/Application\ Support/com.apple.TCC/ ~/Library/Safari` in their own Terminal and sends both error texts with whether Terminal has the grant (the release has no Debug menu), and the next fix is chosen from them: the system database step above, or another folder

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->


**What changed.** The app's `FullDiskAccessProbe` and the CLI's `report::full_disk_access` now try, in order, the per-user `TCC.db` (opened) and `~/Library/Safari` (listed); any one read means granted, any error of a step sends the check to the next, none read means not granted. One list in each side (`FULL_DISK_ACCESS_PROBES`, `FullDiskAccessProbe.homeRelative`), held equal by a DDCKit contract test. The Debug menu logs each step's errno. PR #123 (5b001c1).

**Why.** On macOS 27 the per-user `TCC.db` is gone or refused even with the grant, so the popover said "Full Disk Access Needed" with the switch on, and every background scan skipped the prompting areas (the owner, 2026-11-05).

**Tests.** ddc-core unit tests over temporary homes (an unreadable probe is a file in place of its folder, so they run on every platform); a background-scan CLI test with only `Library/Safari`; DDCKit `FullDiskAccessProbeTests` with mode-000 refusals and the contract test. DDCKit 493 passed, Windows leg 1014/1014, Linux gate-fast green, a Debug build of the app target compiled, PR and main Gates green.

**Risks and follow-ups.** Not yet seen on the owner's macOS 27: whether `~/Library/Safari` lists there with the grant (owner check in the runbook; if not, the system database step with a `--system-root` for `ddc scan` is next). An account where Safari never ran keeps today's answer. Older macOS keeps its answer; the app and the CLI need macOS 26 or later. The Gate builds no app target (a follow-up for the build matrix). `bench/make_baseline.py` still reads only `TCC.db`.

## Technical Notes

- Apple gives no API for "does this process have Full Disk Access"; every check is a read of something only the grant opens (DTS in forum thread 841091 says the same). The list may need another step when a later macOS moves `~/Library/Safari`; the list keeps such a change local.
- No ADR changes: neither "Menu bar app: Swift shell over the ddc CLI" nor "Platform seam: one platform module per OS with declared capabilities" names the probed file; the README's "Full Disk Access" section and the schemas' `full_disk_access` text speak of protected data in general. Closed stories that name `TCC.db` are history and stay.
- The Gate's `app` job runs `make app-test` (DDCKit's `swift test`), not a build of the app target: `AppDelegate.swift` and `DebugMenu.swift` are compiled once by a Debug `xcodebuild` before the push, and the Release build at the release.
- The app and the CLI need macOS 26 or later (`build-app.sh` checks the CLI's minimum, 26.0), so macOS 25 does not run this code at all.
- `bench/make_baseline.py` still reads only `TCC.db` for its header's access line; a bench note, not the product, left for later.
- A listing of `~/Library/Safari` without the grant fails with `EPERM` and shows no prompt (Full Disk Access is never asked for, only granted in Settings).
- Reports written on macOS 27 before this fix say `full_disk_access: false`; the first `ddc changes` across the fix says the home was read differently (`different-access`). Once only.
- The scheduled scan cannot run on the owner's Mac until the story "Scheduled scans: register the agent again after an update" is done; the owner check's report may come from a Scan Now.

## Dependencies

- Story "Menu bar popover" (DDC-002): the live probe, not the report's `full_disk_access`.
- Story "Scheduled background scan" (DDC-001): `PROMPTING_AREAS` skipped without the grant.

## Attachments

-

---

*Last updated: 2026-11-05*

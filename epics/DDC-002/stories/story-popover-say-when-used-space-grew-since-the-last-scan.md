---
type: story
id: "story-popover-say-when-used-space-grew-since-the-last-scan"
epic: "DDC-002"
title: "Popover: say when used space grew since the last scan"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-03
updated: 2026-11-02
external_refs: {}
tags: [app, cli, growth, popover]
code_refs: []
specs: []
started_at: null
closed_at: "2026-11-02T22:03:03.327Z"
plan_updated_at: null
---

# Popover: say when used space grew since the last scan

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

On 2026-11-02 the owner's disk lost about 200 GB in a day to coding agents' build folders, and nobody noticed: the app showed the new folders as rows, but no number it puts forward moved and nothing drew the eye. The owner asked for the app to notice cheaply that used space grew, and to say so.

This is the first story of the research [[research/what-grew-since-the-last-scan-a-ux-concept-for-the-macos-app|What grew since the last scan: a UX concept for the macOS app]] (S5 there). It says *that* used space grew and by how much, without a scan; later stories say *where* (S1, S2, S4). The owner agreed (2026-11-02) to start with this one and to the menu bar badge it lights.

- A new CLI command, `ddc space [--json] [--data-dir DIR]`, reads the free space of the home folder's volume with `statfs` (one syscall, no walk, no lock), compares it with the free space the saved report recorded for that volume, and says whether in-use space grew, or the disk is nearly full. It records a sample in `space.json` and changes nothing else: a call from a terminal or a coding agent never changes what the app shows. The rule lives in Rust, as ADR "Menu bar app: Swift shell over the ddc CLI" §1 requires.
- The app calls it on the hourly tick, on wake, when the popover opens, and at most once per ten minutes from its refresh. The first time a saved report sees `grew` or `low-disk`, the app sets its growth badge (shown when `ShowMenuBarBadge` is on), and the popover shows one band: "↑ 198 GiB more in use on the startup disk than at the last scan, 26 h ago. A scan can tell where. [Scan Now]". Opening the popover clears the badge, and so does a new baseline. Nothing is scanned, notified or cleaned by itself.

## Decomposition

- [x] Core: a `space` module: read `statfs` for the home folder's volume through the existing `StatFs` seam (`report/volumes.rs`); read only the head of `last-scan.json` (`generated_at` and `volumes`, any schema version that has them, as `last_scan_files` reads a head), never its findings; apply the rule; append a sample to `space.json` atomically (`{schema_version, samples: [{at, mount_point, total, free}]}`, at most one an hour for 14 days, plus the latest; a corrupt or missing file, or another `schema_version`, reads as empty). Nothing else is written — evidence: `crates/ddc-core/src/space.rs` (`decide`, `check`, `record`, the head read); `crates/ddc-core/tests/space.rs`
- [x] The rule: the baseline is the saved report's volume whose `mount_point` is the home's; none → `baseline: null`: `low-disk` under 10 % free, else `quiet`. `grew` when in-use grew by at least max(20 GiB, 5 % of `total_bytes`); `low-disk` when free is under 10 % of `total_bytes` and `notify_low_disk` (settings) is not false, which wins over `grew`; otherwise `quiet`. Shrinking never fires — evidence: `space.rs::decide`; `tests/space.rs::the_threshold_is_five_percent_of_the_volume_with_20_gib_at_least`, `under_ten_percent_free_is_low_disk_whatever_the_delta_and_grew_is_still_said`, `no_saved_report_or_none_of_the_homes_volume_gives_no_baseline`
- [x] Baseline time: `Volume` gains an optional `read_at` (RFC 3339), set only by the `clean --items` edit when it refreshes that volume's `free_bytes` (`after_clean::finish`); a scan leaves it absent (its volumes are read with `generated_at`); `ddc disk free`'s edit leaves it. `SpaceReport.baseline` carries `source: scan | clean` and the matching time. Additive field: no schema bump (`model.rs`: "New optional fields keep SCHEMA_VERSION"); `docs/scan.schema.json` regenerated — evidence: `model.rs` `Volume.read_at`; `report/after_clean.rs` `finish`; `cmd/clean_items.rs`; `tests/after_clean.rs::only_a_volume_the_clean_read_again_takes_its_read_time`; `clean_items_cli.rs::a_clean_of_a_list_dates_the_volume_and_space_compares_with_it`; `docs/scan.schema.json`
- [x] CLI: `ddc space` with `Command::Space` in `Cli::prints_json()`; text: one sentence for each state (grew; low disk; quiet: "No more in use on the startup disk than at the last scan."; no report: the low-disk sentence when under 10 % free, else "No saved scan to compare with yet: run `ddc scan`."); `--json`: `docs/space.schema.json` with its own `schema_version`, `{schema_version, checked_at, volume {mount_point, total_bytes, free_bytes}, baseline {source, at, free_bytes} | null, in_use_delta (signed bytes; null when baseline is null), grew (in-use reached the threshold, also when `low-disk` wins; false when `baseline` is null), state}`; exit 0, or 1 with a `ddc:` line when `statfs` fails, or 69 on an unsupported platform; never 75; an unreadable `settings.json` is a `ddc: warning:` and low disk then counts; a failed write of `space.json` is a warning and exit 0; a cancelled report still serves as the baseline; `--data-dir` and `DDC_DATA_DIR` honoured — evidence: `crates/ddc-cli/src/cmd/space.rs` and its tests (`each_state_says_one_sentence`, `a_volume_that_cannot_be_read_is_an_error_and_prints_nothing`, `an_unreadable_settings_file_warns_and_low_disk_counts`, `a_samples_file_that_cannot_be_written_is_a_warning`); `cli.rs`, `gate.rs`; `docs/space.schema.json`
- [x] ADR "Menu bar app: Swift shell over the ddc CLI": §6 item 13 (`ddc space`, its schema, `space.json`, exit codes 0/1/69, never 75); §5's badge gains the growth badge as a second source, with its own clear rule (the popover opens, or the baseline changes) — evidence: the ADR's amendment of 2026-11-03 (§5 "A second source"; §6 item 13)
- [x] README: the command; `space.json` under "Where things live" — evidence: README "`ddc space`" section, "Where things live" row, and the menu bar app bullet
- [x] DDCKit: `SpaceReport` decoded strictly; a `.space` `CLIRunner.Kind` counted in `RunnerActivity.scans`, queued like `rules`; the gate `scan == nil && scheduledTask == nil && !cleaning && !updateWaiting` checked before enqueueing (a waiting update's relaunch waits for the runner to go idle) — evidence: `SpaceReport.swift`; `CLIRunner.Kind.space`; `RunnerActivity.scans`; `PopoverModel.checkSpace` gate; `SpaceTests.strict`, `SpaceTests.gating`, `RunnerActivityTests`
- [x] App: the calls (hourly tick, a `--scheduled` launch included; `NSWorkspace.didWakeNotification`, new; popover open; from refresh at most once per 10 minutes by the injected clock). "Once per baseline" is the app's: a `GrowthBadgedFor` key in `UserDefaults` holds the baseline (mount point, source, time) the badge was set for, or `{mount point, baseline: null}` for a low disk with no saved report yet, once until a report appears; the flag is set whatever `ShowMenuBarBadge` says and `IconState` gates the display, as `AttentionBadgePending` does, and is read where `badgePending` is read on refresh (a `--scheduled` launch writes it); it clears when the popover opens and when the current baseline (from `ddc space` or a reloaded report) no longer matches `GrowthBadgedFor`; on the hourly tick `space` is called after the tick's scheduled scan, not before; the band shows only while the `SpaceReport`'s baseline is the loaded report's (the home volume's `read_at`, else `generated_at`), and the app calls `space` again after any scan or clean it ran; before the first result, or while a scan runs, there is no band; `IconState.Mark.badge` and VoiceOver name "used space grew" (and both reasons when both hold) — evidence: `ScheduledWork.runTick`; `StatusItemController` (`popoverOpened()`, the wake observer, `popoverDidClose`); `AppPreferences` `GrowthBadgePending`/`GrowthBadgedFor`; `IconState`; `SpaceTests.badgeOncePerBaseline`, `SpaceTests.throttle`, `ScheduledScanTests.spaceAfterTheTick`; `ContractTests.spaceCallers`
- [x] Popover: a band between the header and the main block. Grew: "↑ <delta> more in use on <volume> than at the last scan, <age>." / "than after your clean, <age>." plus "A scan can tell where." and Scan Now, enabled by `canScan`; grew while scans are paused: the number only; low disk (new UI, warning ink): "<volume> is N% full" (`Wording.volume`, the percentage rounded down as `NotificationText` does), plus the growth clause only when the CLI says `grew: true` (no comparison in Swift); quiet: no band, with or without a baseline; no baseline and low disk: the low-disk band without the growth clause. Ages through `Wording.age`; `Wording.volume` in mid-sentence lower-cases only its leading "The" ("the startup disk"; a volume named "Work" stays "Work") — evidence: `GrowthBand.make`; `GrowthBandView` in `PopoverView.swift`; `SpaceTests.bandPerState`, `SpaceTests.wording`; a low disk that also grew offers the scan too (a reviewer's point, decided 2026-11-03)
- [x] Owner checks in the runbook "Owner checks: manual acceptance across stories": the badge and the band after a real build; on this Mac, `ddc space`'s free bytes against Finder's "Available" and System Settings → Storage, the difference written into Technical Notes (purgeable space and snapshots) — evidence: the runbook "Owner checks: manual acceptance across stories", section "Used space grew (DDC-002)"
- [x] The research note follows the story: its S1 row (`read_at` moved to S5) and its `ddc space` section and `SpaceReport` sketch (no `first_for_baseline`, `fired_for` or `written_at`; `settled` stays as S6's) — evidence: the note's "Amended 2026-11-03" line in its `ddc space` section

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

- [x] Test (core, fake `StatFs`, a saved report in a temp data dir): on a 500 GiB volume, 24 GiB more in use than the baseline is `quiet` and 25 GiB is `grew` (the boundary, 5 % of `total_bytes`), with `in_use_delta` the signed bytes and `grew` true exactly when the delta reaches the threshold, also when `low-disk` wins; on a 100 GiB volume 19 GiB is `quiet` and 20 GiB `grew`; free under 10 % is `low-disk` whatever the delta; with `notify_low_disk: false`, under 10 % free is `grew` when the delta reaches the threshold and `quiet` otherwise; more free space than the baseline is `quiet` — evidence: `crates/ddc-core/tests/space.rs::the_threshold_is_five_percent_of_the_volume_with_20_gib_at_least`, `under_ten_percent_free_is_low_disk_whatever_the_delta_and_grew_is_still_said`; `cmd/space.rs::an_unreadable_settings_file_warns_and_low_disk_counts`
- [x] Test: no saved report, or a report with no volume whose `mount_point` is the home's, gives `baseline: null` and `quiet`, or `low-disk` when under 10 % free; other volumes of the report are ignored; a report of an older schema version with `generated_at` and `volumes` still serves as the baseline — evidence: `tests/space.rs::no_saved_report_or_none_of_the_homes_volume_gives_no_baseline`, `a_report_of_an_older_schema_version_or_a_cancelled_one_still_serves`
- [x] Test: `space.json` keeps at most one sample per hour for 14 days plus the latest, drops older ones, is written atomically, and a corrupt or missing file reads as empty; `ddc space` writes nothing else (the saved report's bytes and mtime unchanged) — evidence: `tests/space.rs::space_json_keeps_the_first_sample_of_each_hour_for_14_days_and_the_newest`, `a_sample_from_the_future_or_without_a_time_leaves`, `a_missing_corrupt_or_other_version_samples_file_reads_as_empty`, `the_check_writes_nothing_and_the_record_only_space_json_atomically`; `space_cli.rs::space_json_compares_with_the_scan_and_writes_only_its_samples`
- [x] Test: the baseline is `source: scan` at `generated_at` for a scanned report; after a `clean --items` edit the edited volume has `read_at` and the baseline is `source: clean` at that time; a `ddc disk free` edit leaves `read_at` absent and the baseline `scan` — evidence: `tests/space.rs::the_baseline_is_the_scan_or_the_clean_that_read_the_free_space`; `tests/after_clean.rs::only_a_volume_the_clean_read_again_takes_its_read_time`; `clean_items_cli.rs::a_clean_of_a_list_dates_the_volume_and_space_compares_with_it`
- [x] Test (CLI): `ddc space --json` validates against the committed `docs/space.schema.json`, which equals the generated one (`committed_*_schema_is_current`, `DDC_UPDATE_SCHEMA`); the text output for each state is its one sentence; exit 1 with a `ddc:` line when the volume cannot be read; `--data-dir` is honoured; it runs while another process holds the lock — evidence: `tests/space.rs::committed_space_schema_is_current`; `space_cli.rs` (`contract::holds("space")`, `space_says_one_sentence_and_reads_ddc_data_dir`, `space_runs_while_a_clean_holds_the_lock`); `cmd/space.rs::a_volume_that_cannot_be_read_is_an_error_and_prints_nothing`
- [x] Test (DDCKit): `SpaceReport` decodes a fixture the CLI wrote and refuses an unknown `state`; `ContractTests` compares the schema's `state` values with `SpaceReport.State.allCases`; the popover model gives each state its band text and actions (`in_use_delta` null with no baseline) (grew with Scan Now following `canScan`; paused: number only; low disk with and without growth; quiet, with or without a baseline: none; null baseline and low disk: the low-disk band without the growth clause), with ages through `Wording.age` — evidence: `SpaceTests.decodes`, `SpaceTests.strict`, `SpaceTests.bandPerState`; `ContractTests.spaceContracts`
- [x] Test (DDCKit): the first `grew` or `low-disk` for a baseline sets the growth badge flag and records `GrowthBadgedFor`; a second call with the same baseline does not set it again; a new baseline does; a `ddc space` run from elsewhere between two app calls does not stop the app's badge (the CLI keeps no once-per-baseline state); opening the popover clears it, and so does a reloaded report or `space` result with another baseline (a scheduled scan or a Scan Now from the menu between the signal and the popover leaves no badge without a band); on the hourly tick `space` runs after the scheduled scan; with `ShowMenuBarBadge` off the flag is set but not shown; the notification badge keeps its own rule; after a Scan Now or a clean `--items` edit, the band of the old baseline is gone at once, before the next `space` result — evidence: `SpaceTests.badgeOncePerBaseline`, `SpaceTests.lowDiskWithoutReport`, `SpaceTests.badgePreference`, `SpaceTests.newBaselineClears`, the Scan Now and scheduled-scan tests (the old band and badge dropped as soon as the report is read), the popover-open test (no badge for a band on screen)
- [x] Test (DDCKit): no `space` call is enqueued while a scan, a scheduled task or a clean runs; calls happen on the scheduled tick, on wake, on popover open, and from refresh at most once per 10 minutes by the injected clock — evidence: `SpaceTests.gating`, `SpaceTests.throttle`; `ScheduledScanTests.spaceAfterTheTick`
- [x] `make gate-fast` and `make ddckit-test` pass, and the Gate's `app` job; tests that cannot run on Windows or Linux are listed in `.config/native-skips/` — evidence: `make gate-fast` 867/867 in WSL and `swift test` 482/482 on the Mac on the merged branch; main's Gate run 41529900566 on 19dbe51 (0.28.1) green, the `app` job included; `make dist` built, signed and notarized the app with the band; the new Windows-skipping tests listed in `.config/native-skips/windows.txt`

## Final Summary

**What changed.** A new command, `ddc space [--json]`, reads the home folder's volume with `statfs` and compares it with the free space the saved report recorded for that volume (`read_at` after a `clean --items` edit, else the scan's time). It says `grew` at max(20 GiB, 5 % of the volume), `low-disk` under 10 % free unless `notify_low_disk` is off, else `quiet`; it walks nothing, takes no lock and writes only its samples (`space.json`). The menu bar app calls it on the hourly tick after any scheduled scan, on wake, when the popover opens and at most every ten minutes; the first `grew` or `low-disk` for a baseline lights the badge (the app keeps "once per baseline", so a terminal run cannot consume it), and the popover shows one band with Scan Now. The band shows only for the loaded report's baseline; a new scan drops it at once. Contract: ADR "Menu bar app: Swift shell over the ddc CLI" §5 "A second source" and §6 item 13. Shipped in 0.28.0 and published as 0.28.1 (with a musl build fix of 0.27.0's btrfs code).

**Why.** On 2026-11-02 the owner's disk lost about 200 GB in a day to agents' build folders, and nothing the app shows moved. This is S5 of the research "What grew since the last scan: a UX concept for the macOS app": it says *that* space grew; S1, S2 and S4 will say *where*.

**Tests.** Rust: `tests/space.rs`, `cmd/space.rs`, `space_cli.rs`, `after_clean.rs`, the end-to-end `clean_items_cli.rs` test, the schema and fixture tests (867/867 in WSL). DDCKit: `SpaceTests`, `ContractTests.spaceContracts`/`spaceCallers`, `IconStateTests`, `RunnerActivityTests`, `ScheduledScanTests` (482/482). Main's Gate green on 0.28.1, and `make dist` compiled the app target, which CI does not.

**Decided on the way.** The critic moved "once per baseline" from the CLI to the app, made `low-disk` need no saved report, and replaced a threshold field with a `grew` flag so Swift compares nothing; the planner added the update-waiting gate and exit 69; the reviewer added the popover-visible rule (no badge for a band on screen) and, by the owner's interest in the 200 GB case, a low disk that also grew offers the scan too.

**Risks and follow-ups.**
- Owner checks: the badge and band after a real build, and `ddc space` against the Finder's "Available" (purgeable space and snapshots count as in use here); in the runbook.
- A `ddc disk free` or a terminal `ddc clean` leaves the saved report's free space stale until the next scan, so growth up to what it freed reads as quiet (S1 fixes the terminal clean).
- Next: S1 (keep the previous report), S2 (`ddc changes`), S4 (the "Grew since …" entry in Review).
- The full Gate (with the musl cross-check) runs only on main, so 0.27.0's musl error surfaced after merge; worth a story to run `cross-check` on PRs that touch `platform/linux`.

## Technical Notes

- `statfs`'s `f_bavail` counts only unconditionally free space: purgeable space and blocks of local snapshots count as in use (Eclectic Light, 2023-04-27, "Where does macOS get its volume free space figures from?"). The scan stores the same reading, so the signal and the scan agree; the Finder's larger "Available" is not used. The owner check records how far apart they are on a real Mac.
- The volume is the home folder's (on macOS the Data volume); the band names it through `Wording.volume` ("the startup disk"), never "your home folder".
- Once per baseline is presentation, so it lives in the app: the CLI stays read-only apart from its samples, and a terminal call cannot consume the app's badge.
- The hourly tick exists only while "Scan automatically" is on; with it off, the app checks only while it runs (wake, popover, refresh).
- Until S1, a terminal `ddc clean` leaves its pre-clean rescan as the saved report, so growth up to what that clean freed reads as quiet until the next scan.
- A `ddc disk free` that frees space does not touch the report's volumes, so growth up to what it freed reads as quiet until the next scan.
- Privacy: `space.json` holds a mount point, sizes and times only, in the data folder; nothing leaves the Mac.
- Windows and Linux: the command is platform-neutral behind the `StatFs` seam (`statvfs` on Linux; Windows later); their apps follow later.
- Not in this story: where the growth went (S1, S2, S4); the settle rule and a scan or notification triggered by growth (S6, which needs an ADR §5 amendment for notifications).

## Dependencies

- [[research/what-grew-since-the-last-scan-a-ux-concept-for-the-macos-app|What grew since the last scan: a UX concept for the macOS app]]: the concept; this is its S5
- ADR "Menu bar app: Swift shell over the ddc CLI": §1 (rules in Rust), §5 (the badge is not a notification), §6 (item 13 added here)
- [[epics/DDC-002/stories/story-background-scan-and-notifications-in-the-app|Background scan and notifications in the app]]: the hourly tick and the badge this story reuses

## Attachments

-

---

*Last updated: 2026-11-02*

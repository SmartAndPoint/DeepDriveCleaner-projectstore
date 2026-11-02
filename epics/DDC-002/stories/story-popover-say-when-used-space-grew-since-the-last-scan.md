---
type: story
id: "story-popover-say-when-used-space-grew-since-the-last-scan"
epic: "DDC-002"
title: "Popover: say when used space grew since the last scan"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-03
updated: 2026-11-03
external_refs: {}
tags: [app, cli, growth, popover]
code_refs: []
specs: []
started_at: null
closed_at: null
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
- The app calls it on the hourly tick, on wake, when the popover opens, and at most once per ten minutes from its refresh. The first time a saved report sees `grew` or `low-disk`, the app sets its growth badge (shown when `ShowMenuBarBadge` is on), and the popover shows one band: "↑ 198 GiB more in use on the startup disk than at the last scan, 26 h ago. A scan can tell where. [Scan Now]". Opening the popover clears the badge, and so does a new saved report. Nothing is scanned, notified or cleaned by itself.

## Decomposition

- [ ] Core: a `space` module: read `statfs` for the home folder's volume through the existing `StatFs` seam (`report/volumes.rs`); read only the head of `last-scan.json` (`generated_at` and `volumes`, any schema version that has them, as `last_scan_files` reads a head), never its findings; apply the rule; append a sample to `space.json` atomically (`{schema_version, samples: [{at, mount_point, total, free}]}`, at most one an hour for 14 days, plus the latest; a corrupt or missing file, or another `schema_version`, reads as empty). Nothing else is written
- [ ] The rule: the baseline is the saved report's volume whose `mount_point` is the home's; none → `baseline: null`, `quiet`. `grew` when in-use grew by at least max(20 GiB, 5 % of `total_bytes`); `low-disk` when free is under 10 % of `total_bytes` and `notify_low_disk` (settings) is not false, which wins over `grew`; otherwise `quiet`. Shrinking never fires
- [ ] Baseline time: `Volume` gains an optional `read_at` (RFC 3339), set only by the `clean --items` edit when it refreshes that volume's `free_bytes` (`after_clean::finish`); a scan leaves it absent (its volumes are read with `generated_at`); `ddc disk free`'s edit leaves it. `SpaceReport.baseline` carries `source: scan | clean` and the matching time. Additive field: no schema bump (`model.rs`: "New optional fields keep SCHEMA_VERSION"); `docs/scan.schema.json` regenerated
- [ ] CLI: `ddc space` with `Command::Space` in `Cli::prints_json()`; text: one sentence for each state (grew; low disk; quiet: "No more in use on the startup disk than at the last scan."; no report: "No saved scan to compare with yet: run `ddc scan`."); `--json`: `docs/space.schema.json` with its own `schema_version`, `{schema_version, checked_at, volume {mount_point, total_bytes, free_bytes}, baseline {source, at, free_bytes} | null, in_use_delta (signed bytes; null when baseline is null), state}`; exit 0, or 1 with a `ddc:` line when `statfs` fails; never 75; `--data-dir` and `DDC_DATA_DIR` honoured
- [ ] ADR "Menu bar app: Swift shell over the ddc CLI": §6 item 13 (`ddc space`, its schema, `space.json`); §5's badge sentence gains the growth badge as a second source, with its own clear rule (the popover opens). README: the command; `space.json` under "Where things live"
- [ ] DDCKit: `SpaceReport` decoded strictly; a `.space` `CLIRunner.Kind` counted in `RunnerActivity.scans`, queued like `rules`; the gate `scan == nil && scheduledTask == nil && !cleaning` checked before enqueueing
- [ ] App: the calls (hourly tick, a `--scheduled` launch included; `NSWorkspace.didWakeNotification`, new; popover open; from refresh at most once per 10 minutes by the injected clock). "Once per baseline" is the app's: a `GrowthBadgedFor` key in `UserDefaults` holds the baseline (mount point, source, time) the badge was set for; the flag is set whatever `ShowMenuBarBadge` says and `IconState` gates the display, as `AttentionBadgePending` does, and is read where `badgePending` is read on refresh (a `--scheduled` launch writes it); it clears when the popover opens and when the current baseline (from `ddc space` or a reloaded report) no longer matches `GrowthBadgedFor`; on the hourly tick `space` is called after the tick's scheduled scan, not before; `IconState.Mark.badge` and VoiceOver name "used space grew" (and both reasons when both hold)
- [ ] Popover: a band between the header and the main block. Grew: "↑ <delta> more in use on <volume> than at the last scan, <age>." / "than after your clean, <age>." plus "A scan can tell where." and Scan Now, enabled by `canScan`; grew while scans are paused: the number only; low disk (new UI, warning ink): "<volume> is N% full" (`Wording.volume`, the percentage rounded down as `NotificationText` does), plus the growth clause only when the delta reaches the `grew` threshold; quiet or no baseline: no band. Ages through `Wording.age`; `Wording.volume` lower-cased in mid-sentence
- [ ] Owner checks in the runbook "Owner checks: manual acceptance across stories": the badge and the band after a real build; on this Mac, `ddc space`'s free bytes against Finder's "Available" and System Settings → Storage, the difference written into Technical Notes (purgeable space and snapshots)
- [ ] The research note follows the story: its S1 row (`read_at` moved to S5) and its `ddc space` section and `SpaceReport` sketch (no `first_for_baseline`, `fired_for`, `settled` or `written_at`)

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

- [ ] Test (core, fake `StatFs`, a saved report in a temp data dir): on a 500 GiB volume, 24 GiB more in use than the baseline is `quiet` and 25 GiB is `grew` (the boundary, 5 % of `total_bytes`), with `in_use_delta` the signed bytes; on a 100 GiB volume 19 GiB is `quiet` and 20 GiB `grew`; free under 10 % is `low-disk` whatever the delta; with `notify_low_disk: false`, under 10 % free is `grew` when the delta reaches the threshold and `quiet` otherwise; more free space than the baseline is `quiet`
- [ ] Test: no saved report, or a report with no volume whose `mount_point` is the home's, gives `baseline: null` and `quiet`; other volumes of the report are ignored; a report of an older schema version with `generated_at` and `volumes` still serves as the baseline
- [ ] Test: `space.json` keeps at most one sample per hour for 14 days plus the latest, drops older ones, is written atomically, and a corrupt or missing file reads as empty; `ddc space` writes nothing else (the saved report's bytes and mtime unchanged)
- [ ] Test: the baseline is `source: scan` at `generated_at` for a scanned report; after a `clean --items` edit the edited volume has `read_at` and the baseline is `source: clean` at that time; a `ddc disk free` edit leaves `read_at` absent and the baseline `scan`
- [ ] Test (CLI): `ddc space --json` validates against the committed `docs/space.schema.json`, which equals the generated one (`committed_*_schema_is_current`, `DDC_UPDATE_SCHEMA`); the text output for each state is its one sentence; exit 1 with a `ddc:` line when the volume cannot be read; `--data-dir` is honoured; it runs while another process holds the lock
- [ ] Test (DDCKit): `SpaceReport` decodes a fixture the CLI wrote and refuses an unknown `state`; `ContractTests` compares the schema's `state` values with `SpaceReport.State.allCases`; the popover model gives each state its band text and actions (`in_use_delta` null with no baseline) (grew with Scan Now following `canScan`; paused: number only; low disk with and without growth; quiet and null: none), with ages through `Wording.age`
- [ ] Test (DDCKit): the first `grew` or `low-disk` for a baseline sets the growth badge flag and records `GrowthBadgedFor`; a second call with the same baseline does not set it again; a new baseline does; a `ddc space` run from elsewhere between two app calls does not stop the app's badge (the CLI keeps no once-per-baseline state); opening the popover clears it, and so does a reloaded report or `space` result with another baseline (a scheduled scan or a Scan Now from the menu between the signal and the popover leaves no badge without a band); on the hourly tick `space` runs after the scheduled scan; with `ShowMenuBarBadge` off the flag is set but not shown; the notification badge keeps its own rule
- [ ] Test (DDCKit): no `space` call is enqueued while a scan, a scheduled task or a clean runs; calls happen on the scheduled tick, on wake, on popover open, and from refresh at most once per 10 minutes by the injected clock
- [ ] `make gate-fast` and `make ddckit-test` pass, and the Gate's `app` job; tests that cannot run on Windows or Linux are listed in `.config/native-skips/`

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- `statfs`'s `f_bavail` counts only unconditionally free space: purgeable space and blocks of local snapshots count as in use (Eclectic Light, 2023-04-27, "Where does macOS get its volume free space figures from?"). The scan stores the same reading, so the signal and the scan agree; the Finder's larger "Available" is not used. The owner check records how far apart they are on a real Mac.
- The volume is the home folder's (on macOS the Data volume); the band names it through `Wording.volume` ("the startup disk"), never "your home folder".
- Once per baseline is presentation, so it lives in the app: the CLI stays read-only apart from its samples, and a terminal call cannot consume the app's badge.
- The hourly tick exists only while "Scan automatically" is on; with it off, the app checks only while it runs (wake, popover, refresh).
- Until S1, a terminal `ddc clean` leaves its pre-clean rescan as the saved report, so growth up to what that clean freed reads as quiet until the next scan.
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

*Last updated: 2026-11-03*

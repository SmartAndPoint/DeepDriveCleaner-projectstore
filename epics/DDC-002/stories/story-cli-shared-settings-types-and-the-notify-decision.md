---
type: story
id: "story-cli-shared-settings-types-and-the-notify-decision"
epic: "DDC-002"
title: "CLI: shared settings, types and the notify decision"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-18
external_refs: {design_canvas: "https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3"}
tags: [cli, contract, json, settings, notifications]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# CLI: shared settings, types and the notify decision

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The app and the terminal must scan the same folders with the same types, and the notify rule must stay in one place. This story gives the CLI:
- a shared settings file;
- `ignored` findings for switched-off types and small items;
- the rule groups the app's Types tab lists;
- free space per scanned volume;
- the notify decision as JSON, and the commands that record it;
- a `schedule install` that respects the app's scheduler.

It is §5 (the scheduler record, the notify state) and §6 items 5 to 9 of the ADR "Menu bar app: Swift shell over the ddc CLI".

## Decomposition

Rough until the plan gate.

- [ ] `<data-dir>/settings.json` with `schema_version`:
  - `roots`, `excludes`, `disabled_rules`, `min_age_days`, `min_size`, `notify_threshold`, `notify_low_disk`, `scheduler`;
  - read by `scan`, `clean` and the background scan; a flag wins over the file;
  - a malformed file is an error that names it, never silent defaults;
  - the CLI writes it only to clear a stale `scheduler` record
- [ ] `ignored` findings: a rule in `disabled_rules`, or a size under `min_size`, gives `ignored: true` and `selected: false`, with no `not_selected` entry. Not counted in the reclaimable totals or the notify decision. `min_size` from the settings is a scan-time floor; the `--min-size` flag of `clean` stays a plan filter; a plan leaves out items under the larger of the two (ADR §6 item 7)
- [ ] A listed item whose rule is in `disabled_rules` is still cleaned by `clean --items`: the list is the user's confirmation, and the app never lists `ignored` items
- [ ] `rules --json`: every rule id with its category, class, removal method and a `group` label ("Python environments", "node_modules", "Package caches", … as on the canvas's Types tab)
- [ ] `volumes` in the scan report: free and total bytes of each scan root's volume, once per volume; the low-disk check applies to every scanned volume, not only `statfs(home)`, and names the volume
- [ ] `scan --background --notify json`:
  - stdout `{schema_version, status: "scanned", notify, reason, volume, reclaimable, free, total}`, `reason` one of `threshold`, `low-disk` or null;
  - with the lock held: exit 0 and `{schema_version, status: "skipped", reason: "busy"}`;
  - no notification is sent and nothing is written;
  - a scan that starts before `snooze_until` decides "do not notify";
  - schema in `docs/notify.schema.json`
- [ ] `notify-state.json` has one writer, the CLI; the app calls two hidden subcommands:
  - `notify ack --bytes <n>`: records now and `n` as the last notification, clears `snooze_until`;
  - `notify snooze --until <time>`: records `snooze_until` and clears `last_at` and `last_bytes`, so the first scan after the snooze notifies whenever it is over the threshold or low on space
- [ ] A notifier seam: `notify::send` (osascript) is reached only through a `Notifier` the background command receives, so a test double can record calls
- [ ] `schedule install`:
  - refuses (exit 2) while the `scheduler` record exists and its app path exists, naming the app;
  - ignores a stale record (the path is gone), clears it and says so;
  - when the CLI runs from inside an app bundle, the plist it writes carries `AssociatedBundleIdentifiers` = `com.smartandpoint.deepdrivecleaner`, so Login Items lists it under the app

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test: `roots`, `excludes`, `disabled_rules`, `min_age_days` and `min_size` from `settings.json` apply when no flag is given, a flag overrides the file, and a malformed file is an error naming its path
- [ ] Test: findings of a disabled rule and findings under `min_size` carry `ignored: true` and `selected: false` and no `not_selected`; they are left out of the reclaimable totals and the notify decision; `docs/scan.schema.json` gains only optional fields, so `schema_version` stays 1
- [ ] Test: `clean --items` cleans a listed item whose rule is in `disabled_rules`
- [ ] Test: with `min_size` 50 MB in `settings.json`, `ddc clean --min-size 10M` plans no item under 50 MB, and `--min-size 100M` none under 100 MB
- [ ] Test: every rule id in the catalog has a `group` in `rules --json`; a new rule without one fails the test
- [ ] Test: with two scan roots on different volumes (a temp disk image, or a fake `statfs` port), `volumes` lists both once; a unit test of the decision with the second volume under 10 % free returns `low-disk` naming it
- [ ] Test: `--notify json` output validates against `docs/notify.schema.json` for the threshold, low-disk and no-notify cases, and `notify-state.json` is byte-identical before and after
- [ ] Structural test: with `--notify json`, the `Notifier` double records no call; without it, one call when the decision is to notify
- [ ] Test: with the lock held, `scan --background --notify json` exits 0 and prints `{"status": "skipped", "reason": "busy"}` with its `schema_version`
- [ ] Test: after `notify ack --bytes N`, a decision with the same amount a day later is "do not notify"; after `notify snooze --until T`, a scan starting before T decides "do not notify", and the first scan starting after T over the threshold decides "notify" although the amount grew less than 20 % and less than a week passed
- [ ] Test: `schedule install` with a `scheduler` record whose app path exists exits 2 and names the app; with a record whose path is gone it installs, clears the record and prints a note; run from a copy inside a temp `X.app/Contents/Helpers/`, the plist it writes carries `AssociatedBundleIdentifiers`
- [ ] `make check` passes

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- Versioning: `ignored` and `volumes` are optional fields, so the scan report's `schema_version` stays 1. A new `not_selected` reason would bump it, because `NotSelected` is a tagged enum (`#[serde(tag = "reason")]` in `model.rs`). `settings.json`, the notify decision and the progress events each carry their own `schema_version`.
- `notify::send` runs `/usr/bin/osascript` by absolute path, so taking `osascript` off `PATH` proves nothing; the `Notifier` seam is what the structural test needs.
- 0.1.0 saves the notify state only after `osascript` succeeded (`cmd/background.rs`). `notify ack` keeps that meaning: the state changes only once a notification was shown.
- Preferences only the app uses (the schedule, the size next to the icon, the badge, working hours, the default for `review` items) live in its `UserDefaults`, not in `settings.json` (ADR §6 item 6).
- The canvas's Types tab says "Switching a type off keeps it in the report but never selects it": that is `ignored`.

## Dependencies

- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]
- [[epics/DDC-002/stories/story-cli-clean-an-explicit-item-list|CLI: clean an explicit item list]]

## Attachments

- Design canvas: https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3

---

*Last updated: 2026-10-18*

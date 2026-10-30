---
type: story
id: "story-cli-shared-settings-types-and-the-notify-decision"
epic: "DDC-002"
title: "CLI: shared settings, types and the notify decision"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-19
external_refs: {design_canvas: "https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV"}
tags: [cli, contract, json, settings, notifications]
code_refs: ["README.md", "crates/ddc-cli/src/cli.rs", "crates/ddc-cli/src/cmd/background.rs", "crates/ddc-cli/src/cmd/clean.rs", "crates/ddc-cli/src/cmd/clean_items.rs", "crates/ddc-cli/src/cmd/mod.rs", "crates/ddc-cli/src/cmd/notify.rs", "crates/ddc-cli/src/cmd/rules.rs", "crates/ddc-cli/src/cmd/scan.rs", "crates/ddc-cli/src/cmd/schedule.rs", "crates/ddc-cli/src/platform/launchd.rs", "crates/ddc-cli/src/notify.rs", "crates/ddc-cli/src/prompt.rs", "crates/ddc-cli/src/render.rs", "crates/ddc-cli/tests/background_cli.rs", "crates/ddc-cli/tests/clean_items_cli.rs", "crates/ddc-cli/tests/notify_cli.rs", "crates/ddc-cli/tests/rules_cli.rs", "crates/ddc-cli/tests/settings_cli.rs", "crates/ddc-cli/tests/structure.rs", "crates/ddc-core/src/exec/preflight.rs", "crates/ddc-core/src/lib.rs", "crates/ddc-core/src/model.rs", "crates/ddc-core/src/notify.rs", "crates/ddc-core/src/plan.rs", "crates/ddc-core/src/report/mod.rs", "crates/ddc-core/src/report/volumes.rs", "crates/ddc-core/src/rules/catalog.rs", "crates/ddc-core/src/rules/listing.rs", "crates/ddc-core/src/rules/mod.rs", "crates/ddc-core/src/settings.rs", "crates/ddc-core/tests/notify.rs", "crates/ddc-core/tests/report.rs", "crates/ddc-core/tests/rules.rs", "crates/ddc-core/tests/settings.rs", "docs/notify.schema.json", "docs/rules.schema.json", "docs/scan.schema.json", "docs/settings.schema.json"]
specs: []
started_at: "2026-10-19T05:30:25.087Z"
closed_at: "2026-10-19T06:20:32.271Z"
plan_updated_at: "2026-10-19T05:30:25.087Z"
---

# CLI: shared settings, types and the notify decision

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | done |
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

- [x] `<data-dir>/settings.json` with `schema_version`:
  - `roots`, `excludes`, `disabled_rules`, `min_age_days`, `min_size`, `notify_threshold`, `notify_low_disk`, `scheduler`;
  - read by `scan`, `clean` and the background scan; a flag wins over the file, and an `--exclude` that drops folders the file protects names them first (asks in a terminal, refuses otherwise; a scan only warns);
  - a settings root or exclude that does not exist is skipped with a note; all settings roots missing is an error;
  - a malformed file is an error that names it, never silent defaults;
  - the CLI writes it only to clear a stale `scheduler` record
- [x] `ignored` findings: a rule in `disabled_rules`, or a size under `min_size`, gives `ignored: true` and `selected: false`, keeping the `not_selected` it would have had, so the app applies a type switched back on at once. Not counted in the reclaimable totals or the notify decision. `min_size` from the settings is a scan-time floor; the `--min-size` flag of `clean` stays a plan filter; a plan leaves out items under the larger of the two (ADR §6 item 7)
- [x] A listed item whose rule is in `disabled_rules` is still cleaned by `clean --items`: the list is the user's confirmation, and the app never lists `ignored` items
- [x] `rules --json`: every rule id with its category, class, removal method, whether the Types tab can switch it off (`switchable`) and a `group` label ("Python environments", "node_modules", "Package caches", … as on the canvas's Types tab)
- [x] `volumes` in the scan report: free and total bytes of each scan root's volume, once per volume; the low-disk check applies to every scanned volume, not only `statfs(home)`, fires only when something selected lies on that volume, and names it
- [x] `scan --background --notify json`:
  - stdout `{schema_version, status: "scanned", notify, reason, volume, reclaimable, free, total}`, `reason` one of `threshold`, `low-disk` or null;
  - with the lock held: exit 0 and `{schema_version, status: "skipped", reason: "busy"}`;
  - no notification is sent and `notify-state.json` is not written (`last-scan.json` and `background.log` still are);
  - a scan that starts before `snooze_until` decides "do not notify";
  - schema in `docs/notify.schema.json`
- [x] `notify-state.json` has one writer, the CLI; the app calls two hidden subcommands:
  - `notify ack --bytes <n>`: records now and `n` as the last notification, clears `snooze_until`;
  - `notify snooze --until <time>`: records `snooze_until` and clears `last_at` and `last_bytes`, so the first scan after the snooze notifies whenever it is over the threshold or low on space
- [x] A notifier seam: `notify::send` (osascript) is reached only through a `Notifier` the background command receives, so a test double can record calls
- [x] `schedule install`:
  - refuses (exit 2) while the `scheduler` record exists and its app path exists, naming the app;
  - ignores a stale record (the path is gone), clears it and says so;
  - when the CLI runs from inside an app bundle, the plist it writes carries `AssociatedBundleIdentifiers` = `com.smartandpoint.deepdrivecleaner`, so Login Items lists it under the app

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

No spec covers this story. Its contract is §5 and §6 items 5–9 of the ADR "Menu bar app: Swift shell over the ddc CLI". The plan builds on "CLI: clean an explicit item list", as committed in 81233c7.

**Owner decisions (2026-10-19).**
- `--exclude` replaces the excludes from `settings.json`, as every other flag replaces its setting. When that drops folders the settings protect, ddc first names exactly those folders:
  - `clean` in a terminal asks `[y/N]` (`prompt::yes`: `y` or `yes` goes on); `--yes` does not answer this question. Any other answer ends it as a declined final confirmation does: "Nothing was changed.", exit 0;
  - `clean` without a terminal, and `clean --items` (dry run included, so the sheet's prediction matches the clean), refuse with exit 2 and name them;
  - `scan`, foreground or background, and plain `clean --dry-run` change nothing, so they only warn: on stderr, or in `background.log`;
  - the question comes after the settings are loaded and before the lock and the scan, so a declined answer takes no lock and writes no `last-scan.json`.
- The low-disk notification fires for a volume only when that volume is under 10 % free **and** a selected finding lies on it. 0.1.0 had the same gate ("ddc clean cannot help"), applied to the home volume only.

**Decisions taken in the plan.**
- **Precedence.**
  - A flag wins over the file: positional paths over `roots`, `--exclude` over `excludes` (with the confirmation above), `--min-age` over `min_age_days`, `--threshold` over `notify_threshold`.
  - `disabled_rules`, the scan-time `min_size` and `notify_low_disk` come from the file only.
  - The `--min-size` flag of `clean` is the plan filter of ADR §6 item 7, never an override: a plan leaves out items under the larger of the two.
- **Missing paths.** A settings root or exclude that does not exist (an unplugged `/Volumes/Work`) is skipped, with a note on stderr or in `background.log`. If every settings root is missing, that is an error naming them; it never falls back to home silently. An empty or absent `roots` means home, as today. Flags stay strict, as today.
- **Dropped excludes.** With `--exclude` given, the dropped folders are the settings excludes that exist (canonicalised, as `report::scan` canonicalises excludes), are neither equal to nor under a flag exclude (the visitor keeps everything under an exclude), and touch an effective root (`e.starts_with(root) || root.starts_with(e)`). Only those are named. `scan_options` stats paths, so it is tested with a temp tree.
- **Unknown ids.** An unknown id in `disabled_rules` is inert, so a rule renamed in a later release never breaks a scan. Unknown keys in the file are an error (`deny_unknown_fields`): app-only preferences never belong there, and an unknown key may be a protection this CLI does not know, which it must not silently ignore. The CLI ships inside the app bundle, so the two change together; a new key comes with a `schema_version` bump.
- **Defaults.** `notify_low_disk` defaults to `true`, the behaviour of 0.1.0.
- **`min_size`** compares a finding's `bytes`, as the `--min-size` plan filter does (`plan.rs`).
- **ddc's own tombstones are never ignored.** `ddc.tombstone` (`.ddc-deleting-*` left by an interrupted clean) stays selectable under any `min_size` and `disabled_rules`; otherwise the leftover would stay on disk for good.
- **`ignored` keeps the default-selection reason.** An ignored finding keeps the `not_selected` it would have had, so the app can apply a type switched back on at once (ADR §6 item 7, "applies the same rule to the loaded report at once"). `ignored` takes precedence: `selected` is false whatever `not_selected` says. This changes the story's earlier wording, "no `not_selected` entry", and the ADR is revised to match.
- **Totals.** Ignored findings are left out of every total (class counts and bytes, and the selected totals), but stay in `findings`. `Totals` gains optional `ignored_count` and `ignored_bytes` (skipped when 0), so the sums still reconcile and the popover can say how much the Types settings hide. The terminal report marks them `ignored` and counts them on one trailing line.
- **Plans.** `build_plan` leaves out every ignored finding. `plan_listed` never reads selection, so a listed item of a disabled rule is still cleaned.
- **The decision JSON.** `reason` is null exactly when `notify` is false; when both reasons apply, `low-disk` wins. `volume`, `free` and `total` describe the volume the decision is about: for `low-disk`, the triggering volume (under 10 % free **and** carrying a selected finding; the lowest free ratio among several); otherwise the volume of the first entry of `scan_roots` (the home volume when home is scanned). `reclaimable` is the report's selected reclaimable bytes, all volumes together. The optional fields are always written, as null when absent.
- **Volumes `statfs` cannot read** (a share unmounted during the scan) are left out of `volumes` with a note; low-disk cannot fire for them.
- **Snooze time.** `notify snooze --until` takes RFC 3339 UTC with `Z` (`humantime::parse_rfc3339` accepts no offsets); anything else exits 2. The snooze is compared with the scan's start (`env.now()`).
- **`--notify json` and `--no-notify`** conflict (clap).
- **The 0.1.0 agent's threshold.** `schedule install` writes `--threshold` into the plist only when the flag is given, so a CLI-only user's `notify_threshold` in `settings.json` applies.
- **Locks.** `notify ack` and `notify snooze` take no lock: their writes are atomic, and a clean's exclusive lock must not block an ack.
- **What `--notify json` writes.** It writes no `notify-state.json` and sends nothing. `last-scan.json` and `background.log` are still written, because the app reads the scan after a scheduled run.

**Placement.**
- `crates/ddc-core/src/settings.rs` (new):
  - `Settings` (`deny_unknown_fields`, `schema_version` 1) with `SchedulerRecord { by: "app", app }`;
  - `SettingsError { Io, Malformed }`, which always names the path;
  - `load(&Env)`, where a missing file gives the defaults, and a `schema_version` other than 1 is an error naming the path and the version, checked before the fields are decoded;
  - `clear_scheduler(&Env)`, through `store::write_atomic`;
  - one `scan_options(&self, &ScanFlags) -> (ScanOptions, notes, dropped_excludes)`, used by every command. It stats paths (missing roots and excludes, dropped excludes).
- `crates/ddc-core/src/model.rs`:
  - `Finding.ignored` (`default`, skipped when false);
  - `Volume { mount_point, roots, free_bytes, total_bytes }` and `ScanReport.volumes` (`default`, skipped when empty);
  - `From<&Removal> for RemovalKind`.
  - `docs/scan.schema.json` is regenerated; `schema_version` stays 1.
- `crates/ddc-core/src/report/mod.rs`:
  - `ScanOptions` gains `disabled_rules` and `min_size`;
  - `finding()` sets `ignored` and forces `selected = false`;
  - `totals()` skips ignored findings and fills `ignored_count` and `ignored_bytes`;
  - `scan_with(env, opts, progress, &dyn StatFs)` sits behind `scan`, as `walk` sits on `walk_with_reader`.
- `crates/ddc-core/src/report/volumes.rs` (new): a `StatFs` port, `SystemStatFs` over `sys::statfs`, and `volumes()`, one entry per mount point with the roots on it. The walker never crosses filesystems, so a finding lies on its root's volume.
- `crates/ddc-core/src/plan.rs`: `build_plan` filters out ignored findings.
- `crates/ddc-core/src/rules/`:
  - `Rule.group` (`&'static str`) and `GROUPS` in `catalog.rs`, in the order of the canvas's Types tab;
  - `listing.rs` (new): `RuleList { schema_version, groups, rules: [RuleInfo { id, group, category, class, removal, listed, switchable }] }`. `switchable` is false for `ddc.tombstone` and for every rule `plan::action_for` gives no action (`user.kept` and the other user-data rules, `Removal::None` such as the Trash, `NativeOnly` with no command such as the VM runtimes); the app shows a switch only for groups with switchable rules.
  - Groups, the canvas's ten in its order, then two without a switch:
    - **Python environments:** venv, tox.
    - **node_modules:** both rules.
    - **Build outputs:** target dirs, Gradle, CMake, JS build caches, `__pycache__`, the per-project tool caches (`.pytest_cache`, `.mypy_cache`, `.ruff_cache`), CocoaPods `Pods`.
    - **Package caches:** `cache.*` except Colima, the Maven repo, test browsers.
    - **IDE caches.**
    - **Xcode data.**
    - **App caches:** with `CACHEDIR.TAG` caches.
    - **Old toolchains.**
    - **ML models.**
    - **VM and container images:** with the Colima cache.
    - **Interrupted cleanups:** `ddc.tombstone`, not switchable.
    - **Your data:** the user-data rules and the Trash, not switchable.
- `crates/ddc-core/src/notify.rs` (new): the decision's contract. It is `NotifyReport { schema_version, status }`, where the status is either `scanned`, with `{ notify, reason, volume, reclaimable, free, total }`, or `skipped` with `{ reason: busy }`. The JSON contract lives in core because the CLI is a binary only, and tests decode its output through core types.
- `crates/ddc-cli/src/notify.rs`, where the rule stays:
  - `NotifyState.snooze_until` (`default`), so 0.1.0 state files still load;
  - `Facts` from the report, per volume;
  - `decide(&Facts, &Config, &NotifyState) -> Decision` checks, in order: snoozed, low-disk, threshold, then the 0.1.0 repeat rule, unchanged;
  - a `Notifier` trait and `Osascript`; `send` becomes private;
  - `ack` and `snooze`.
- `crates/ddc-cli/src/cmd/background.rs`:
  - `run_with(args, &Env, &dyn Notifier, &dyn StatFs, out)`. The dispatcher in `cmd/scan.rs` constructs `Osascript` and `SystemStatFs` and passes them in, so `background.rs` never names them;
  - a held lock with `--notify json` prints `skipped`/`busy` and exits 0;
  - in json mode it prints the decision and never calls the notifier or saves the state;
  - otherwise it notifies, then calls `ack` only after a notification was shown.
- `crates/ddc-cli/src/cli.rs`:
  - `--min-age` (scan and clean), `--threshold` on `scan --background`, and `--threshold` on `schedule install` become `Option`, so an absent flag is visible; `install_with` and its "notify from" line follow;
  - a hidden `--notify json` on `scan --background`;
  - hidden `notify ack --bytes N` and `notify snooze --until T`;
  - a visible `rules [--json]`.
- `crates/ddc-cli/src/cmd/`: settings wired into `scan.rs`, `clean.rs`, `clean_items.rs` (its dry run too) and `background.rs`; the exclude confirmation in `clean.rs`; new `rules.rs` and `notify.rs`.
- `crates/ddc-cli/src/render.rs`: `status()` prints `ignored` for an ignored finding; the category and class lines count only findings that are not ignored; one trailing line "N items ignored by settings.json (size)".
- `crates/ddc-cli/src/cmd/schedule.rs` and `launchd.rs`:
  - `install_with(interval, threshold, exe: &Path, env: &Env, lc: &dyn Launchctl, out) -> anyhow::Result<ExitCode>`; `run` wraps it with the real `Env`, `current_exe` and `SystemLaunchctl`;
  - a live `scheduler` record exits 2 naming the app, before `launchd::install` and any launchctl call;
  - a stale record is cleared, with a note;
  - `app_bundle_of(exe)` matches `X.app/Contents/Helpers/<bin>` and adds `AssociatedBundleIdentifiers` to the plist as an `<array>` of one `<string>com.smartandpoint.deepdrivecleaner</string>`, the form `SMAppService` writes. `procs::app_bundle` is left alone: widening it would change the owner-running guard.
- `docs/notify.schema.json`, `docs/settings.schema.json` and `docs/rules.schema.json` (new), one contract each with its own `schema_version`, each with a committed-schema test.
- README: `ddc rules`, `settings.json` (location, fields, precedence, the exclude confirmation), and the `schedule install` refusal.

**Order.**
1. Model fields, then the scan schema; spike the schemars flatten for `NotifyReport` first.
2. `ignored` in the report, the totals and `build_plan`. `two_scans_give_the_same_report` must ignore live `volumes`.
3. `volumes` with the `StatFs` port.
4. `settings.rs` and its schema.
5. CLI flags to `Option`; settings wired into every command; the exclude confirmation.
6. Rule groups, `listing.rs`, `ddc rules`.
7. The notify contract, the decision and the `Notifier` seam.
8. `background.rs` with `--notify json` and the busy JSON.
9. `notify ack` / `snooze`.
10. `schedule install`.
11. README, then `make check`.

**Tests per criterion.**
- AC1: `crates/ddc-core/tests/settings.rs` covers precedence, missing paths and every malformed shape. `crates/ddc-cli/tests/settings_cli.rs` checks end to end, for `scan`, `clean --dry-run` and `scan --background --no-notify`, that the file applies, that a flag wins, and that a malformed file exits 1 naming its path.
- AC2: `tests/report.rs` checks the ignored fields, the kept `not_selected`, the totals with `ignored_count`, a tombstone never ignored, the schema's `required` list, and that a 0.1.0 report still decodes. `scan_cli.rs` checks the `ignored` status and the trailing line. `background_cli.rs` checks that disabling the only reclaimable rule gives no notification.
- AC3: `clean_items_cli.rs`.
- AC4: `clean_cli.rs`, with fixtures of about 30, 70 and 110 MiB.
- AC5: `tests/rules.rs` `every_rule_has_a_group`; `rules_cli.rs` round-trips the output through `RuleList`.
- AC6: a fake `StatFs` in `tests/report.rs` (two volumes, three roots, one volume `statfs` cannot read), and a unit test of `decide` that names the triggering volume, not the lowest one. The tests never mount a disk image.
- AC7: `background_cli.rs` with `notify_low_disk: false`, because a live disk may itself be low; the low-disk case is a unit test of `run_with` with the fake port; the committed schema in `tests/notify.rs`.
- AC8: a recording `Notifier` in a unit test of `run_with`; `structure.rs` forbids `Osascript`, `Command` and `/usr/bin/` in `background.rs`.
- AC9: the busy test in `background_cli.rs`.
- AC10: unit tests of `decide` with explicit times; `notify_cli.rs` for the state file effects and a bad `--until`.
- AC11: unit tests in `schedule.rs` and `launchd.rs` with `FakeLaunchctl`, a temp `Env` and a dummy `X.app/Contents/Helpers/deepdrivecleaner`; the plist passes `plutil -lint`. No test runs `launchctl`: the owner's 0.1.0 agent has the same label.
- Exclude criterion:
  - `crates/ddc-core/tests/settings.rs` computes the dropped folders over a temp tree (equal, under a flag exclude, outside the roots, a symlinked spelling);
  - `clean_cli.rs` checks the refusal without a terminal and with `--items`, and the warning on `scan` and `clean --dry-run`;
  - the `[y/N]` runs under `/usr/bin/script`, as the Ctrl-C test of the previous story does.

**Pitfalls.**
- `disabled_rules` must not reach the visitor or the catalog. Filtering there would change claims, break `Catalog::verify`, and make `plan_listed` refuse listed items as `rule-mismatch`.
- `structure.rs` keeps `exec::`, `plan::`, `trash` and `Consent` out of `background.rs`, `notify.rs` and `launchd.rs`.
- Unit tests of `run_with` live in the binary crate (`#[cfg(test)]`); it has no library target.
- The app must no longer pass roots and excludes: scan, clean and the dry run resolve them from `settings.json`. A settings change between review and clean can make items `changed-since-scan` or `kept`. "Review and clean window" carries this in its Technical Notes, and the previous story's pitfall is marked superseded.
- `ScanReport.swift` decodes strictly, while `ignored` is absent when false and `volumes` when empty: the popover and review stories decode both with `decodeIfPresent`.
- The home volume's mount point is `/System/Volumes/Data`. The app needs a display name for it; the CLI does not.
- `clear_scheduler` rewrites the whole file while the app may write it too. A stale record means the app is gone, so the race needs a bundle moved while it runs; noted, not fixed.

**Proposed `code_refs`.** `crates/ddc-core/src/settings.rs`, `crates/ddc-core/src/notify.rs`, `crates/ddc-core/src/model.rs`, `crates/ddc-core/src/report/`, `crates/ddc-core/src/plan.rs`, `crates/ddc-core/src/rules/`, `crates/ddc-core/src/exec/preflight.rs`, `crates/ddc-core/src/lib.rs`, `crates/ddc-core/tests/`, `crates/ddc-cli/src/cli.rs`, `crates/ddc-cli/src/cmd/`, `crates/ddc-cli/src/notify.rs`, `crates/ddc-cli/src/launchd.rs`, `crates/ddc-cli/src/render.rs`, `crates/ddc-cli/tests/`, `docs/scan.schema.json`, `docs/notify.schema.json`, `docs/settings.schema.json`, `docs/rules.schema.json`, `README.md`.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [x] Test: `roots`, `excludes` and `min_age_days` from `settings.json` apply when no flag is given, and positional paths, `--exclude` and `--min-age` win over them, as `--threshold` wins over `notify_threshold`; `disabled_rules`, `min_size` and `notify_low_disk` come from the file only; a settings root or exclude that does not exist is skipped with a note, all settings roots missing is an error naming them, and an empty `roots` means home; a malformed file is an error naming its path — evidence: crates/ddc-core/tests/settings.rs::{no_file_and_an_empty_roots_list_give_home, the_file_applies_and_a_flag_wins, missing_settings_folders_are_skipped_and_all_roots_missing_is_an_error, a_malformed_file_is_an_error_naming_it}; crates/ddc-cli/tests/settings_cli.rs::{the_settings_apply_and_a_flag_wins (scan and the background scan), a_malformed_settings_file_or_no_existing_root_is_an_error_naming_it (scan, clean and the background scan, --notify json included)}; crates/ddc-cli/src/cmd/background.rs::{the_threshold_flag_wins_over_the_settings, low_disk_can_be_switched_off_in_the_settings}
- [x] Test: an `--exclude` that drops folders `settings.json` protects names exactly those folders (an `--exclude` above a settings exclude, and a settings exclude outside the scanned roots, are not named): `ddc clean` in a terminal asks `[y/N]` before taking the lock, and anything but `y` or `yes` ends with "Nothing was changed." and the tree untouched; `ddc clean` without a terminal and `clean --items` (dry run included) exit 2 naming them; `scan` and `clean --dry-run` print a warning naming them and run — evidence: crates/ddc-core/tests/settings.rs::an_exclude_flag_names_only_the_protected_folders_it_drops; crates/ddc-cli/tests/settings_cli.rs::an_exclude_flag_that_drops_a_protected_folder_names_it_first (no terminal with and without --yes, --items and its dry run; a pseudo-terminal answers n and y)
- [x] Test: findings of a disabled rule and findings under `min_size` carry `ignored: true` and `selected: false` and keep the `not_selected` they would have had; a `ddc.tombstone` is never ignored; the terminal report marks ignored findings and counts them on one line; they are left out of the reclaimable totals and the notify decision; `docs/scan.schema.json` gains only optional fields, so `schema_version` stays 1 — evidence: crates/ddc-core/tests/report.rs::{ignored_findings_keep_their_reason_and_leave_the_totals_and_the_plan, a_report_without_the_new_fields_still_decodes_and_they_stay_optional}; crates/ddc-cli/tests/settings_cli.rs::switched_off_types_and_small_items_are_ignored (the note line, and the mark under --all; the class sections leave them out, see Deviations); crates/ddc-cli/tests/background_cli.rs::switched_off_types_never_reach_the_notify_decision
- [x] Test: `clean --items` cleans a listed item whose rule is in `disabled_rules` — evidence: crates/ddc-cli/tests/clean_items_cli.rs::a_listed_item_of_a_switched_off_type_is_still_cleaned
- [x] Test: with `min_size` 50 MB in `settings.json`, `ddc clean --min-size 10M` plans no item under 50 MB, and `--min-size 100M` none under 100 MB — evidence: crates/ddc-cli/tests/settings_cli.rs::the_min_size_setting_is_a_floor_under_the_min_size_flag; crates/ddc-core/tests/report.rs::ignored_findings_keep_their_reason_and_leave_the_totals_and_the_plan
- [x] Test: every rule id in the catalog has a `group` in `rules --json`, and `switchable` is false exactly for `ddc.tombstone` and the rules ddc never acts on (`plan::action_for` gives no action: user data, removal `none`, a tool-only removal with no command); a new rule without a group fails the test — evidence: crates/ddc-core/tests/rules.rs::{every_rule_has_a_group, committed_rules_schema_is_current}; crates/ddc-cli/tests/rules_cli.rs::rules_json_lists_every_rule_with_its_group
- [x] Test: with scan roots on two volumes (a fake `statfs` port; no disk image is mounted), `volumes` lists each volume once with its roots; a unit test of the decision with the second volume under 10 % free returns `low-disk` naming it when a selected finding lies on it, and does not notify for it when none does; with the first volume at 3 % free and no selected finding on it, the decision is `low-disk` and `volume` names the second volume — evidence: crates/ddc-core/tests/report.rs::volumes_list_each_volume_once_with_its_roots; crates/ddc-cli/src/notify.rs::low_disk_names_the_volume_where_something_can_be_freed
- [x] Test: `--notify json` output round-trips through `NotifyReport`, and the committed `docs/notify.schema.json` equals the generated one, for the threshold, low-disk (through the fake `statfs` port) and no-notify cases, and `notify-state.json` is byte-identical before and after — evidence: crates/ddc-cli/tests/background_cli.rs::notify_json_prints_the_decision_and_leaves_the_state_alone; crates/ddc-cli/src/cmd/background.rs::a_volume_low_on_space_notifies_and_is_named; crates/ddc-core/tests/notify.rs::{the_decision_and_the_skip_have_the_documented_shape, committed_notify_schema_is_current}
- [x] Structural test: with `--notify json`, the `Notifier` double records no call; without it, one call when the decision is to notify — evidence: crates/ddc-cli/src/cmd/background.rs::{json_prints_the_decision_and_never_notifies_or_records_it, without_json_it_notifies_once_and_records_it}; crates/ddc-cli/tests/structure.rs::the_background_scan_runs_no_program_itself
- [x] Test: with the lock held, `scan --background --notify json` exits 0 and prints `{"status": "skipped", "reason": "busy"}` with its `schema_version` — evidence: crates/ddc-cli/tests/background_cli.rs::background_scan_steps_aside_while_the_lock_is_held
- [x] Test: after `notify ack --bytes N`, a threshold decision (no volume low on space) with the same amount a day later is "do not notify" (low disk repeats since 2026-10-19, see After the close); after `notify snooze --until T`, a scan starting before T decides "do not notify", and the first scan starting after T over the threshold decides "notify" although the amount grew less than 20 % and less than a week passed — evidence: crates/ddc-cli/src/notify.rs::{ack_then_snooze_then_after_the_snooze, a_0_1_0_state_file_still_loads}; crates/ddc-cli/tests/notify_cli.rs::{ack_records_the_notification_and_snooze_clears_it, a_time_that_is_not_utc_rfc_3339_is_refused}
- [x] Test: `schedule install` with a `scheduler` record whose app path exists exits 2 and names the app; with a record whose path is gone it installs, clears the record and prints a note; with its executable path inside a temp `X.app/Contents/Helpers/`, the plist it writes carries `AssociatedBundleIdentifiers`; these tests use a fake launchctl and never run `launchctl` — evidence: crates/ddc-cli/src/cmd/schedule.rs::{a_live_app_scheduler_refuses_without_touching_launchd, a_stale_record_is_cleared_and_the_agent_installed, installed_from_inside_the_app_it_is_listed_under_the_app}; crates/ddc-cli/src/launchd.rs::{generated_plist_passes_plutil, an_agent_installed_from_the_app_is_listed_under_it}
- [x] `make check` passes — evidence: `PATH=$HOME/.cargo/bin:$PATH make check`: 177 tests, fmt and clippy clean (2026-10-19)

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

**What changed.**
- `settings.json` (`crates/ddc-core/src/settings.rs`, schema `docs/settings.schema.json`):
  - The file is strict: unknown keys are refused, and `schema_version` is checked before the fields are decoded.
  - `Settings::resolve` is the one place where a command's flags and the file meet:
    - a flag wins;
    - `disabled_rules`, `min_size` and `notify_low_disk` come from the file only;
    - a missing settings root or exclude is skipped with a note;
    - all settings roots missing is an error that names them.
  - `resolve` also computes the settings excludes an `--exclude` drops.
  - `clear_scheduler` is the CLI's only write to the file.
  - `scan`, `clean`, `clean --items` (its dry run too) and the background scan all use `resolve`.
- The exclude confirmation (owner's decision):
  - `clean` asks `[y/N]` before the lock and the scan; a declined answer ends with "Nothing was changed.";
  - it refuses (exit 2) without a terminal or with `--items`, naming the folders, the generic terminal refusal included;
  - `scan` and `clean --dry-run` only warn.
- `ignored` findings (`model.rs`, `report/mod.rs`):
  - a switched-off rule or a size under `min_size` gives `ignored: true` and `selected: false`, and the finding keeps its `not_selected`;
  - `ddc.tombstone` is never ignored;
  - ignored findings are left out of the class and selected totals and are counted in `Totals.ignored_count` and `ignored_bytes`;
  - `build_plan` skips them, while `plan_listed` still cleans a listed one;
  - the terminal report gives them one note line, and lists them with `--all` marked `ignored`.
- `volumes` in the scan report, through a `StatFs` port (`report/volumes.rs`, `report::scan_with`).
- `ddc rules [--json]` (`rules/listing.rs`, `cmd/rules.rs`, schema `docs/rules.schema.json`): one `GROUPS` table in the order of the canvas's Types tab, then "Interrupted cleanups" and "Your data". `switchable` is false for `ddc.tombstone` and for every rule `plan::action_for` gives no action.
- The notify decision:
  - `crates/ddc-cli/src/notify.rs` decides per volume, checking in order: the snooze, low disk (only a volume under 10 % free with something selected on it; owner's decision), the threshold, then the 0.1.0 repeat rule, since 2026-10-19 for the threshold only (see After the close);
  - the JSON contract is `ddc_core::notify` (schema `docs/notify.schema.json`); `volume`, `free` and `total` describe the volume under 10 % free with something selected on it (also while a snooze holds the notification back), or else the first scan root's volume, or else null;
  - `scan --background --notify json` prints the decision and never notifies or writes the state; a held lock prints `skipped`/`busy` and exits 0;
  - osascript is reached only through a `Notifier` that `cmd/scan.rs` hands in.
- Hidden `notify ack --bytes` and `notify snooze --until` (RFC 3339 UTC); the CLI stays the state's one writer, and neither takes a lock.
- `schedule install` (`cmd/schedule.rs::install_with`, `launchd.rs`):
  - it refuses, naming the app, while the app's `scheduler` record points at an existing bundle, and clears a stale record;
  - from inside `X.app/Contents/Helpers/` it adds `AssociatedBundleIdentifiers`;
  - it writes `--threshold` into the plist only when the flag is given.
- README: `ddc rules`, `clean --items`, the exit codes and a Settings section.

**Why.** This is §5 and §6 items 5–9 of the ADR "Menu bar app: Swift shell over the ddc CLI": the app and the terminal scan the same folders with the same types, and the notify rule stays in one place, in Rust.

**Deviations from the plan.**
- The group labels live in one `GROUPS` table in `rules/listing.rs`, not in a `Rule.group` field. `every_rule_has_a_group` still fails for a rule with no group.
- The terminal report leaves ignored findings out of the class sections and prints one note line; `--all` lists them, marked.
- The end-to-end checks the plan placed in `scan_cli.rs` and `clean_cli.rs` live in `settings_cli.rs`.
- The plan's `scan_options` is `Settings::resolve`, returning `Resolved { options, missing, dropped_excludes }`; `SettingsError` gained `NoRoots`.

**Tests.**
- `make check`: 177 tests pass, fmt and clippy clean (2026-10-19).
- New or extended for this story:
  - core: `tests/settings.rs` (7), `tests/notify.rs` (2), `tests/report.rs` (3 new), `tests/rules.rs` (2 new);
  - CLI integration: `settings_cli.rs` (5), `notify_cli.rs` (3), `rules_cli.rs` (1), `background_cli.rs` (2 new and the busy JSON), `clean_items_cli.rs` (1 new), `structure.rs` (1 new);
  - unit tests: `cmd/background.rs` (5), `cmd/schedule.rs` (3), `notify.rs` (6) and `launchd.rs` (1 new).
- No test runs `launchctl`, `osascript` or `hdiutil`. Live free space never decides a test: tests use `notify_low_disk: false` or a fake `StatFs`.
- `projectstore:reviewer` found no code defect. Its findings were fixed:
  - test gaps: `--threshold` over `notify_threshold`, the `ignored` mark under `--all`, the refusal naming the app;
  - the decision's volume now comes from the first scan root, never from another root standing in;
  - a foreground scan notes a root whose volume cannot be read;
  - the generic terminal refusal names dropped excludes;
  - wider end-to-end checks for the background scan and the plan's group counts;
  - ADR §5 now says `--notify json` still writes `last-scan.json` and `background.log`.
- `projectstore:critic` on this close found no code defect. Its fixes:
  - tests for `clean --exclude` without a terminal and without `--yes`, an explicit empty `roots`, and all settings roots missing end to end (exit 1, nothing on stdout, `--notify json` included);
  - the `Decision.volume` doc and `docs/notify.schema.json` say the low volume is named also when the notification is held back;
  - the app-side hand-offs moved into the stories that carry them.

**Risks and follow-ups.**
- `settings_cli.rs` writes about 210 MiB of fixtures for the `min_size` test.
- A scan root that is itself a finding is selected but never planned, and it still counts towards the threshold. This was already true in 0.1.0.
- What the app must honour is carried in the Technical Notes of "Menu bar popover", "Review and clean window", "Background scan and notifications in the app" and "Distribution: the 0.2.0 cask with the app":
  - decoding the optional fields with `decodeIfPresent`;
  - passing no roots or excludes;
  - the `Z` time format;
  - the startup-disk name;
  - the settings errors;
  - the strict settings file next to an older CLI.
- `clear_scheduler` rewrites the whole file while the app may also write it. That race needs the bundle to be moved while it runs.
- `schedule install` prints a different "notify from" line when no `--threshold` is given.

**After the close (2026-10-19).** The owner decided that low disk no longer goes through the 0.1.0 repeat rule. An almost full disk with something selected on it notifies at every scheduled scan, one caught up after sleep included, until nothing selected is left on it or the user snoozes. There is no minimum amount or gap, and its acknowledgement is the threshold's repeat baseline. The threshold keeps the rule. ADR "Menu bar app: Swift shell over the ddc CLI" §5 and §6 item 9 record it, and the README's `schedule install` row says so. Tests: `crates/ddc-cli/src/notify.rs::{the_threshold_repeats_only_after_growth_or_a_week, low_disk_notifies_at_every_scan_until_snoozed}`; the latter also pins that low disk switched off, or a low volume with nothing selected on it, leaves the repeat rule in force. `make check`: 178 tests. Checked by `projectstore:critic` and `projectstore:reviewer`.

## Technical Notes

- Versioning: `ignored` and `volumes` are optional fields, so the scan report's `schema_version` stays 1. A new `not_selected` reason would bump it, because `NotSelected` is a tagged enum (`#[serde(tag = "reason")]` in `model.rs`). `settings.json`, the notify decision and the progress events each carry their own `schema_version`.
- `notify snooze --until` takes RFC 3339 UTC with `Z` only (`humantime::parse_rfc3339` rejects offsets): the app formats its time that way.
- `notify::send` runs `/usr/bin/osascript` by absolute path, so taking `osascript` off `PATH` proves nothing; the `Notifier` seam is what the structural test needs.
- 0.1.0 saves the notify state only after `osascript` succeeded (`cmd/background.rs`). `notify ack` keeps that meaning: the state changes only once a notification was shown.
- Preferences only the app uses (the schedule, the size next to the icon, the badge, working hours, the default for `review` items) live in its `UserDefaults`, not in `settings.json` (ADR §6 item 6).
- The canvas's Types tab says "Switching a type off keeps it in the report but never selects it": that is `ignored`.

## Dependencies

- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]
- [[epics/DDC-002/stories/story-cli-clean-an-explicit-item-list|CLI: clean an explicit item list]]

## Attachments

- Design canvas: https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV

---

*Last updated: 2026-10-19*

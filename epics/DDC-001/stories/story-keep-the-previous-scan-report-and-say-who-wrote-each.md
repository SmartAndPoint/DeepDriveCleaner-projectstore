---
type: story
id: "story-keep-the-previous-scan-report-and-say-who-wrote-each"
epic: "DDC-001"
title: "Keep the previous scan report, and say who wrote each"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-03
updated: 2026-11-04
external_refs: {}
tags: [cli, growth, report]
code_refs: ["crates/ddc-core/src/store.rs", "crates/ddc-core/src/model.rs", "crates/ddc-core/src/report/after_clean.rs", "crates/ddc-core/src/report/mod.rs", "crates/ddc-core/src/report/recheck.rs", "crates/ddc-core/src/rules/listing.rs", "crates/ddc-core/src/free/mod.rs", "crates/ddc-core/src/space.rs", "crates/ddc-cli/src/cmd/clean.rs", "crates/ddc-cli/src/cmd/clean_items.rs", "crates/ddc-cli/src/cmd/scan.rs", "crates/ddc-cli/src/cmd/background.rs", "crates/ddc-cli/src/cmd/mod.rs", "crates/ddc-cli/tests/previous_scan_cli.rs", "crates/ddc-cli/tests/clean_items_cli.rs", "crates/ddc-cli/tests/golden_cli.rs", "crates/ddc-core/tests/after_clean.rs", "crates/ddc-loadgen/src/run.rs", "docs/scan.schema.json", "docs/space.schema.json"]
specs: []
started_at: "2026-11-04T11:57:45.339Z"
closed_at: "2026-11-04T12:19:34.259Z"
plan_updated_at: "2026-11-04T11:57:45.339Z"
---

# Keep the previous scan report, and say who wrote each

| Field | Value |
|---|---|
| **Epic** | [DDC-001](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

On 2026-11-02 the owner's disk lost about 200 GB in a day to coding agents' build folders. The story "Popover: say when used space grew since the last scan" (S5, shipped in 0.28) now says *that* used space grew. To say *where*, `ddc` needs something to compare with, and today it keeps nothing: every writer replaces `last-scan.json`, and the report before is gone.

This is S1 of the research [[research/what-grew-since-the-last-scan-a-ux-concept-for-the-macos-app|What grew since the last scan: a UX concept for the macOS app]]. It changes no output a person reads. It keeps one more file and adds a few report fields, so that the next story, "ddc changes: what grew between the last two scans" (S2), can compare two reports and trust the result. The users are S2's: a person whose disk filled up, the menu bar app (S4, later), and a coding agent that shows the person what its builds made.

Three changes:

- **`previous-scan.json`.** The rule, stated once and used everywhere below: *a save by `scan`, `background` or `clean-recheck` moves the report it replaces to `previous-scan.json` when that report is whole: its head reads, it is not `cancelled`, and its home and roots are the default ones (those of a scan without paths). The incoming report's roots do not matter. Every other save (`clean`, `clean-items`, `disk-free`) replaces `last-scan.json` and leaves `previous-scan.json` as it is.* So a clean's edit never pushes the scan before it out, and a project scan never becomes the baseline.
- **Who wrote each report, and when, and with which rules.** New optional report fields: `written_by`, `written_at`, `previous_written_at` and `rules_in_force`.
- **A terminal `ddc clean` edits the saved report after it cleans,** as `ddc clean --items` does. Today it saves its re-check before the clean and never after, so the saved report lists what it just removed until the next scan.

S5 already added `Volume.read_at` (set by the `clean --items` edit) and `ddc space`. This story does not change how either works. The terminal clean's new edit sets `read_at` through the same code, so `ddc space` compares with the clean after a terminal clean too, which S5's Technical Notes left open; the docs that say "only `clean --items`" are updated.

## Decomposition

- [x] **Report fields** (`crates/ddc-core/src/model.rs`, `ScanReport`), all optional and skipped when absent, so `SCHEMA_VERSION` stays 8 (as of 0.30.0; the rule at the top of `model.rs`):
  - `written_by`: a strict kebab-case enum `WrittenBy` for the six saves: `scan`, `background`, `clean-recheck` (a terminal `ddc clean`'s re-check, saved before anything is removed), `clean` (that report edited after the clean), `clean-items`, `disk-free`. A new value later bumps `SCHEMA_VERSION`, as every new variant does, so the set is fixed here.
  - `written_at`: RFC 3339 in UTC to the second (`humantime::format_rfc3339_seconds`), like every other time in the report and the journal; set at every save. `generated_at` stays the scan's start, and an edit never changes it.
  - `previous_written_at`: the `written_at` of the report in `previous-scan.json` when this one was saved; absent when there is none. S2 uses it to see that the two files belong together.
  - `rules_in_force`: the sorted rule ids of the platform's catalog (`rules_of(PLATFORM)`, the ids `ddc rules --json` lists). A type switched off in `settings.json` stays in the list, since its findings are still reported as `ignored`.
  - `docs/scan.schema.json` regenerated (`DDC_UPDATE_SCHEMA=1`).
- [x] **`rules_in_force` at scan time**: filled in `report::scan_inner` (`crates/ddc-core/src/report/mod.rs`), which every walk goes through, `recheck::pruned` included. An edit keeps the list of the report it edits.
- [x] **The store** (`crates/ddc-core/src/store.rs`): `PREVIOUS_SCAN = "previous-scan.json"`, `read_previous_scan`, and `write_last_scan(env, &mut report, writer, default_roots)`. The caller passes the default roots (the roots a scan without paths walks, as `cmd/mod.rs` `own_roots` resolves them); the store compares the outgoing head's `home` and `scan_roots` with them itself. One save, in this order:
  1. **Keep.** For `scan`, `background` and `clean-recheck` only: hard-link `last-scan.json` to a temp name (a copy where the file system refuses links), read the head of that linked copy leniently, any schema version (`home`, `scan_roots`, `cancelled`, `written_at`, `previous_written_at`), and when the rule holds rename it onto `previous-scan.json`; otherwise remove the temp copy. Reading the head from the copy, not from `last-scan.json`, means the stamp below always describes the file that was kept.
  2. **Stamp** by what happened: `written_by`, `written_at`, and `previous_written_at` = the kept report's `written_at` when step 1 kept one; else the outgoing report's own `previous_written_at`, carried forward (also when keeping failed); else absent. The stamp goes into the caller's report (`&mut`), so `ddc scan --json` prints exactly what was saved.
  3. **Write** the temp file and fsync it, then rename it onto `last-scan.json`, as `write_atomic` does.

  `last-scan.json` is never written in place and never missing: it is only ever replaced by a rename. Failing to keep the previous report prints a `ddc: warning:` and never fails the save.
- [x] **The six saves**, each with its writer: `cmd/scan.rs` (`scan`), `cmd/background.rs` (`background`), `cmd/clean.rs` (`clean-recheck` before the clean, and the new `clean` after it), `cmd/clean_items.rs` (`clean-items`, the pruned edit and the full walk alike), and `free::edit_last_scan` in `crates/ddc-core/src/free/mod.rs` (`disk-free`).
- [x] **The terminal clean's edit** (`cmd/clean.rs`): after `exec::execute`, the re-check's report is edited by what the clean did and saved again, `written_by: clean`. It goes through `after_clean::edit`, as `--items` does: deleted and trashed items leave, trashed bytes move into the volume's `trash` finding, a tool-cleaned item is measured again, and each volume the clean touched takes `free_after` and `read_at`. `edit` takes a `CleanReport` today; the plan gate chooses between building one from the plan's selected items and widening `edit` to take the executor's items. After a cooperative cancel the edit is saved with what was done; a crash leaves the re-check, as today. A dry run, a declined prompt and "Nothing selected" save only the re-check, as today.
- [x] **Docs that say only `clean --items` sets `read_at`**, now that a terminal clean does too: the `Volume.read_at` doc in `model.rs` (and so `docs/scan.schema.json`); the `BaselineSource::Clean` doc in `crates/ddc-core/src/space.rs` (and so `docs/space.schema.json`); README "`ddc space`" ("a plain `ddc clean` from a terminal does not set it"); ADR §6 item 9's amendment line, §6 item 13 ("set only by that edit") and §5 "A second source" ("a `clean --items` edit that read the free space again").
- [x] **ADR "Menu bar app: Swift shell over the ddc CLI"**: §6 item 1, "The report after the clean", says the terminal clean edits too. A new §6 item 14, "The previous report", holds:
  - `previous-scan.json` and the rule above, word for word;
  - the four fields and the six `written_by` values;
  - the invariant: `last-scan.json` is never written in place, only replaced by a rename;
  - two scans at once (they share the lock): each may keep whatever `last-scan.json` held when it linked it, and the last rename wins, so one of the two scans' reports can be lost. The pair stays consistent, because the stamp comes from the linked copy. A race that still leaves a mismatch reads in S2 as "out of step";
  - two slots only: a project scan after a Scan Now keeps that Scan Now and drops the report before it (Technical Notes). History is S8's.
- [x] **README**: a "Where things live" row for `previous-scan.json`; the `last-scan.json` row says that `ddc clean` and `ddc clean --items` both edit it after cleaning.
- [x] **Tests that list the data files**: the load harness's `owner_state` (`crates/ddc-loadgen/src/run.rs`) watches `previous-scan.json` too, so a load run still proves it never touched the owner's files.
- [x] **Goldens** (`crates/ddc-cli/tests/golden_cli.rs`): `written_at` and `previous_written_at` normalised as `generated_at` is; `rules_in_force` normalised too (to its count, or left out), or every new rule moves every scan golden. Then `make golden-record`, with a diff that holds only the new fields.
- [x] **The disk cost**: the size of one saved report on the load fixture home, measured once with `ddc-loadgen` and written into the Final Summary as a size and a ratio to the report today (a fixture, not an owner machine; no run times).
- [x] **The research note follows the story**: its amendment of 2026-11-03 (the rotation rule as settled here).

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

The planner's placement plan (2026-11-03) split the story into two pull requests; the gate is recorded here after S1a shipped as 0.31.0, so this section says what was planned and how it went.

**S1a, the store and five saves** (0.31.0):
1. `crates/ddc-core/src/model.rs`: `WrittenBy` and the four optional fields; `rules/listing.rs` `rule_ids`; `report/mod.rs` `scan_inner` fills `rules_in_force`.
2. `report/recheck.rs` `walked_roots`; `crates/ddc-cli/src/cmd/mod.rs` `default_roots` (the normalized roots of a scan without paths) and `not_kept` (the warning).
3. `store.rs`: `PREVIOUS_SCAN`, `read_previous_scan`, `write_last_scan(env, &mut report, writer, default_roots) -> Saved { path, kept }` with `Kept::{Kept, Nothing, Failed}`; link, read the head from the linked copy, rename; then stamp; then write. Unit tests for the rule.
4. Five callers: `cmd/scan.rs`, `cmd/background.rs`, `cmd/clean.rs` (the re-check), `cmd/clean_items.rs`, `free/mod.rs`.
5. Schema regenerated; `golden_cli.rs` `VOLATILE` gains `written_at`, the goldens re-recorded on macOS; `ddc-loadgen` `owner_state` and the `interrupt_cli.rs` temp filter learn the new file; `previous_scan_cli.rs`.

**S1b, the terminal clean's edit** (0.32.0):
1. `report/after_clean.rs`: `edit` widened into a private `edit_items` over `Done { path, bytes, outcome }`, with `edit` (a `CleanReport`, unchanged for `--items`) and `edit_run` (an `ExecReport`, `fresh = None`) on top. No fake `CleanReport`, so `plan.rs` and its coverage floor stay untouched.
2. `cmd/clean.rs`: after `exec::execute`, remeasure the tool-cleaned items, `edit_run`, `write_last_scan(.., WrittenBy::Clean, None)`, all before `print_result` and `signals::reraise`.
3. Docs that said only `clean --items` sets `read_at`: `model.rs`, `space.rs`, both schemas, DDCKit's doc comments, README.
4. Tests: `after_clean.rs` (the run edits as its clean report does), `previous_scan_cli.rs` (AC 8 and nothing selected), `clean_items_cli.rs` (a signal during a terminal clean saves the edit first).

AC 8's "declined prompt" cannot be driven from a CLI test (no terminal); the tests cover the dry run and nothing selected, which share its code (Technical Notes).

## Acceptance Criteria

- [x] Test (store, temp data dir): a `scan`, a `background` and a `clean-recheck` save over a whole, uncancelled report of the default roots each move that report to `previous-scan.json` byte for byte; the new report's `previous_written_at` equals the moved report's `written_at`; no temp file is left. — evidence: store.rs `a_scan_background_or_clean_recheck_save_keeps_a_whole_report_byte_for_byte`
- [x] Test: a `clean`, `clean-items` (its pruned and its full-walk case) or `disk-free` save leaves `previous-scan.json` byte-identical and carries `previous_written_at` forward from the report it replaces. — evidence: store.rs `a_clean_clean_items_or_disk_free_save_keeps_nothing_and_carries_the_pair_forward`; previous_scan_cli.rs `a_clean_of_a_list_keeps_nothing_and_carries_the_pair_forward` (pruned and full walk), `a_terminal_clean_edits_its_recheck_and_keeps_nothing` (`clean`)
- [x] Test: a `scan` save keeps nothing, and `previous-scan.json` stays as it was, when the outgoing report is `cancelled`, of other roots or another home, or has a head that cannot be read (then the new report has no `previous_written_at`). The incoming report's roots do not change the decision: a `ddc scan <project>` over a home report keeps the home report. — evidence: store.rs `a_report_that_is_not_whole_or_not_the_default_ones_is_not_kept`, `the_incoming_reports_roots_do_not_change_what_is_kept`
- [x] Test (CLI): a home scan A, a home scan B, then two `ddc scan <project>` runs: `previous-scan.json` is B (the first project scan kept it; the second found a project report and kept nothing), and the last report's `previous_written_at` is B's `written_at`. — evidence: previous_scan_cli.rs `two_home_scans_then_two_project_scans_keep_the_second_home_scan`
- [x] Test: with a folder in the place of `previous-scan.json`, a `scan` save still writes `last-scan.json`, prints a `ddc: warning:` that names `previous-scan.json`, exits 0, carries `previous_written_at` forward and leaves no temp file. — evidence: previous_scan_cli.rs `a_folder_in_the_place_of_previous_scan_json_is_a_warning_and_the_report_is_saved`; store.rs `a_previous_report_that_cannot_be_replaced_fails_the_keep_not_the_save`
- [x] Test: each of the six saves writes its own `written_by` and a `written_at` to the second; an edit keeps `generated_at`; `ddc scan --json` prints the same `written_by`, `written_at` and `previous_written_at` as the saved file. — evidence: previous_scan_cli.rs (scan, background, clean-recheck, clean, clean-items), disk_free.rs `after_brew_cleanup_the_homebrew_cache_finding_is_measured_again` (disk-free); clean_items_cli.rs `a_signal_during_a_terminal_clean_saves_its_edit_before_ddc_ends`
- [x] Test: `rules_in_force` equals the sorted ids of `ddc rules --json` for the platform, in a full scan and in a pruned re-check's report (`recheck::pruned`); a rule in `disabled_rules` is still listed; an edit keeps the edited report's list. — evidence: report.rs `the_report_lists_the_rules_in_force_a_switched_off_one_too`, recheck.rs `a_pruned_recheck_lists_the_rules_in_force`, after_clean.rs `a_deleted_finding_leaves_with_the_reports_own_figures` (an edit keeps the list)
- [x] Test (CLI, fixture home): `ddc clean --yes` removes the selected `safe` items; afterwards `last-scan.json` lacks them, its totals drop by their bytes, the touched volume has `read_at`, and `written_by` is `clean`; `ddc space --json` then gives `baseline.source: clean`. `ddc clean --dry-run` save the re-check with `written_by: clean-recheck`, which keeps the report before it by the rule. — evidence: previous_scan_cli.rs `a_terminal_clean_edits_its_recheck_and_keeps_nothing`, `a_terminal_clean_with_nothing_selected_saves_only_its_recheck`, `the_background_scan_and_a_cleans_recheck_keep_the_report_before_them` (dry run); after_clean.rs `a_terminal_cleans_run_edits_as_its_clean_report_does`
- [x] Test: `--data-dir` and `DDC_DATA_DIR` put `previous-scan.json` in that folder. — evidence: previous_scan_cli.rs `ddc_data_dir_holds_previous_scan_json`
- [x] Test: the committed `docs/scan.schema.json` and `docs/space.schema.json` equal the generated ones (`committed_*_schema_is_current`), and a report with the new fields validates. — evidence: `committed_json_schema_is_current`, `committed_space_schema_is_current`; `contract::holds` in previous_scan_cli.rs
- [x] The app needs no change: DDCKit's `ScanReport` decodes by explicit coding keys and ignores the new ones; `make ddckit-test` passes against the regenerated schemas. — evidence: `make ddckit-test` on macOS, 482 tests in 49 suites passed on the S1b branch (only DDCKit doc comments changed)
- [x] `make gate-fast` passes, and the Linux and Windows legs; a test that cannot run on one of them is listed in `.config/native-skips/`. — evidence: S1b: `make gate-fast` 907/907 on Linux; native Windows leg 959 of 959 passed, the new nothing-selected test then added to the skip list; .config/native-skips/windows.txt lists the new scanning tests; PR #118, main Gate run 41826577549 green

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

**What changed.** ddc keeps the report before the last one in `previous-scan.json`, by the rule in the Description, and every saved report says who wrote it, when, which report its pair holds, and which rules applied (`written_by`, `written_at`, `previous_written_at`, `rules_in_force`; the scan report stays at schema 8). A terminal `ddc clean` now edits its re-check by what it did, as `clean --items` does, so the saved report and `ddc space` follow a clean from the terminal too. Shipped in two releases: S1a as 0.31.0 (the store, the fields, five saves), S1b as 0.32.0 (the terminal clean's edit, the sixth save, and the docs that said only `clean --items` sets `read_at`). ADR "Menu bar app: Swift shell over the ddc CLI" holds the contract: §6 item 14 "The previous report", the amendment of item 1, and the `read_at` lines of §5 and items 9 and 13.

**Why.** S2 ("ddc changes: what grew between the last two scans") needs two reports it can trust as a pair, and a clean must not push the scan before it out of the baseline.

**Tests executed.** S1a: `make gate-fast` 902/902 on Linux and the full Linux leg, the native Windows leg, `make golden-record` on macOS (only `written_at` and `written_by` in the ten goldens). S1b: `make gate-fast` 907/907 on Linux, the native Windows leg (959 of 959 passed, the new nothing-selected test then added to the skip list), `make ddckit-test` on macOS, and the PR's Gate. The reviewer's pass on S1b found no blocker; its should-fixes (the nothing-selected test, DDCKit's doc comments, the bump) went into the PR.

**Disk cost.** `previous-scan.json` is a byte-for-byte earlier `last-scan.json`, so the data folder holds two scan reports instead of one: about twice the report bytes, nothing else.

**Risks and follow-ups.**
- An exit before the terminal clean's save (a journal write failure mid-run) leaves the re-check, which lists what was removed until the next scan, as for `--items` (ADR §6 item 1).
- After SIGTERM, a tool-cleaned item is measured again before ddc ends; a large tool cache delays the exit by that walk, as in `--items`.
- A CI run once timed out `sigterm_or_sighup_during_a_tool_ends_the_tool_and_ddc_with_the_report` (its one-second budget) on a loaded runner; the rerun passed. The new signal test allows ten seconds.
- Next: S2 `ddc changes`, then S4 in the app.

## Technical Notes

- A declined `[y/N]` prompt saves the re-check too, but no CLI test can drive a prompt (none has a terminal, and `ddc clean` refuses a prompt without one), so the ACs cover `--dry-run` and nothing selected; the declined path shares their code (planner, 2026-11-03).

- **Why a clean's edit does not move the old report.** If the `clean --items` edit kept the report it edits, the next comparison would be "the report before the clean against the report after it", and the rows the person did not clean would drop out of "Grew since …" (S4). With the edit not keeping, the comparison after a clean is still "the last scan before against now", and S2 marks the removed items "cleaned by you" from the journal. A terminal `ddc clean` is a scan and a clean in one: its re-check keeps like a scan, and its edit afterwards does not.
- **Two slots, and an agent's project scan.** The shipped skill tells an agent to run `ddc scan "$PWD" --json`, and that already replaces the app's report (`cmd/mod.rs`, `own_roots`). By the rule, that project scan keeps the home report it replaces. So an agent's project scan after a Scan Now moves that Scan Now into `previous-scan.json`, and the home report before it is gone. The next home scan then compares with the Scan Now, minutes or hours old, and S2 says the age. A second project scan keeps nothing, so a project report never becomes the baseline. Keeping more than two reports is S8. A change of `roots` in Settings makes the next save find "other roots" and keep nothing, and S2 says the roots differ.
- **The saves today** (`origin/main`): `cmd/scan.rs:48`, `cmd/background.rs:117`, `cmd/clean.rs:90`, `cmd/clean_items.rs:179`, `free/mod.rs:358`; this story adds the sixth, the terminal clean's edit. A terminal `ddc clean` saves its re-check also on `--dry-run`, on a declined prompt and when nothing is selected, since the save comes before the plan is shown. Each of those keeps by the rule, like a Scan Now.
- **A crash.** Between keeping and the last rename, a crash leaves the outgoing report in both files, with a `last-scan.json` whose `previous_written_at` is not the kept one's `written_at`. S2 reads that as "out of step" until the next save.
- **Windows.** `std::fs::hard_link` works on NTFS; the copy covers a data folder on a file system without links. Renaming onto a file another process holds open without delete sharing can fail on Windows, as `write_atomic` already can; the warning covers the previous file.
- **Privacy.** `previous-scan.json` holds the same paths as `last-scan.json` (project, worktree and folder names), in the same folder, and nothing leaves the machine. The cask's `zap` already trashes the whole `~/Library/Application Support/DeepDriveCleaner` folder (`packaging/homebrew/deepdrivecleaner.rb.tmpl`), so it covers the new file.
- **After an upgrade** the first scan keeps a report without the new fields. S2 compares only reports that have `written_at` and `rules_in_force`, so growth shows from the second scan after the upgrade.
- **Not in this story:** `ddc changes` and the `ddc scan` block (S2); anything in the app (S4); folder sizes (S3); `ddc disk free` reading the volumes' free space again (it still leaves them as scanned, as S5 notes); keeping the previous whole-disk report (S7); history beyond two reports (S8).
- **Linux and Windows.** The store, the fields and the clean edit are platform-neutral and reach them with the CLI.
- First save in a fresh data folder: a missing `last-scan.json` is "nothing to keep", silently, never a keep failure or a `ddc: warning:` (the fresh-home goldens and cases read stderr).
- `write_last_scan` returns an enum saying whether it kept a report (kept with its `written_at`, kept a report older than this story with none, or kept nothing) beside the `&mut` stamp; `written_at` is `SystemTime::now()` at the save, not `Env::now()` (fixed at process start, so a clean's edit would get the start time); `ddc changes`'s block after `ddc scan` uses it, since `previous_written_at` is also carried forward.
- After a crash between the renames, the pair is out of step until the next *keeping* save (`scan`, `background`, `clean-recheck`); a `clean-items` or `disk-free` save carries the mismatch forward.

## Dependencies

- [[research/what-grew-since-the-last-scan-a-ux-concept-for-the-macos-app|What grew since the last scan: a UX concept for the macOS app]]: the concept; this is its S1
- [[epics/DDC-002/stories/story-popover-say-when-used-space-grew-since-the-last-scan|Popover: say when used space grew since the last scan]]: added `Volume.read_at` and `ddc space`, which this story reuses
- [[epics/DDC-002/stories/story-faster-clean-re-check-only-what-the-selection-needs|Faster clean: re-check only what the selection needs]]: the `clean --items` edit (`after_clean`) that the terminal clean now shares
- [[epics/DDC-001/stories/story-safe-cleanup-ddc-clean|Safe cleanup: ddc clean]]: the terminal clean this story changes
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]: §5 and §6 items 1, 9 and 13 amended, §6 item 14 added
- Needed by: [[epics/DDC-001/stories/story-ddc-changes-what-grew-between-the-last-two-scans|ddc changes: what grew between the last two scans]]

## Attachments

-

---

*Last updated: 2026-11-04*

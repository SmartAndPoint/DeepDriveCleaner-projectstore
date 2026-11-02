---
type: story
id: "story-keep-the-previous-scan-report-and-say-who-wrote-each"
epic: "DDC-001"
title: "Keep the previous scan report, and say who wrote each"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-03
updated: 2026-11-03
external_refs: {}
tags: [cli, growth, report]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
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

- [ ] **Report fields** (`crates/ddc-core/src/model.rs`, `ScanReport`), all optional and skipped when absent, so `SCHEMA_VERSION` stays 7 (the rule at the top of `model.rs`):
  - `written_by`: a strict kebab-case enum `WrittenBy` for the six saves: `scan`, `background`, `clean-recheck` (a terminal `ddc clean`'s re-check, saved before anything is removed), `clean` (that report edited after the clean), `clean-items`, `disk-free`. A new value later bumps `SCHEMA_VERSION`, as every new variant does, so the set is fixed here.
  - `written_at`: RFC 3339 in UTC to the second (`humantime::format_rfc3339_seconds`), like every other time in the report and the journal; set at every save. `generated_at` stays the scan's start, and an edit never changes it.
  - `previous_written_at`: the `written_at` of the report in `previous-scan.json` when this one was saved; absent when there is none. S2 uses it to see that the two files belong together.
  - `rules_in_force`: the sorted rule ids of the platform's catalog (`rules_of(PLATFORM)`, the ids `ddc rules --json` lists). A type switched off in `settings.json` stays in the list, since its findings are still reported as `ignored`.
  - `docs/scan.schema.json` regenerated (`DDC_UPDATE_SCHEMA=1`).
- [ ] **`rules_in_force` at scan time**: filled in `report::scan_inner` (`crates/ddc-core/src/report/mod.rs`), which every walk goes through, `recheck::pruned` included. An edit keeps the list of the report it edits.
- [ ] **The store** (`crates/ddc-core/src/store.rs`): `PREVIOUS_SCAN = "previous-scan.json"`, `read_previous_scan`, and `write_last_scan(env, &mut report, writer, default_roots)`. The caller passes the default roots (the roots a scan without paths walks, as `cmd/mod.rs` `own_roots` resolves them); the store compares the outgoing head's `home` and `scan_roots` with them itself. One save, in this order:
  1. **Keep.** For `scan`, `background` and `clean-recheck` only: hard-link `last-scan.json` to a temp name (a copy where the file system refuses links), read the head of that linked copy leniently, any schema version (`home`, `scan_roots`, `cancelled`, `written_at`, `previous_written_at`), and when the rule holds rename it onto `previous-scan.json`; otherwise remove the temp copy. Reading the head from the copy, not from `last-scan.json`, means the stamp below always describes the file that was kept.
  2. **Stamp** by what happened: `written_by`, `written_at`, and `previous_written_at` = the kept report's `written_at` when step 1 kept one; else the outgoing report's own `previous_written_at`, carried forward (also when keeping failed); else absent. The stamp goes into the caller's report (`&mut`), so `ddc scan --json` prints exactly what was saved.
  3. **Write** the temp file and fsync it, then rename it onto `last-scan.json`, as `write_atomic` does.

  `last-scan.json` is never written in place and never missing: it is only ever replaced by a rename. Failing to keep the previous report prints a `ddc: warning:` and never fails the save.
- [ ] **The six saves**, each with its writer: `cmd/scan.rs` (`scan`), `cmd/background.rs` (`background`), `cmd/clean.rs` (`clean-recheck` before the clean, and the new `clean` after it), `cmd/clean_items.rs` (`clean-items`, the pruned edit and the full walk alike), and `free::edit_last_scan` in `crates/ddc-core/src/free/mod.rs` (`disk-free`).
- [ ] **The terminal clean's edit** (`cmd/clean.rs`): after `exec::execute`, the re-check's report is edited by what the clean did and saved again, `written_by: clean`. It goes through `after_clean::edit`, as `--items` does: deleted and trashed items leave, trashed bytes move into the volume's `trash` finding, a tool-cleaned item is measured again, and each volume the clean touched takes `free_after` and `read_at`. `edit` takes a `CleanReport` today; the plan gate chooses between building one from the plan's selected items and widening `edit` to take the executor's items. After a cooperative cancel the edit is saved with what was done; a crash leaves the re-check, as today. A dry run, a declined prompt and "Nothing selected" save only the re-check, as today.
- [ ] **Docs that say only `clean --items` sets `read_at`**, now that a terminal clean does too: the `Volume.read_at` doc in `model.rs` (and so `docs/scan.schema.json`); the `BaselineSource::Clean` doc in `crates/ddc-core/src/space.rs` (and so `docs/space.schema.json`); README "`ddc space`" ("a plain `ddc clean` from a terminal does not set it"); ADR §6 item 9's amendment line, §6 item 13 ("set only by that edit") and §5 "A second source" ("a `clean --items` edit that read the free space again").
- [ ] **ADR "Menu bar app: Swift shell over the ddc CLI"**: §6 item 1, "The report after the clean", says the terminal clean edits too. A new §6 item 14, "The previous report", holds:
  - `previous-scan.json` and the rule above, word for word;
  - the four fields and the six `written_by` values;
  - the invariant: `last-scan.json` is never written in place, only replaced by a rename;
  - two scans at once (they share the lock): each may keep whatever `last-scan.json` held when it linked it, and the last rename wins, so one of the two scans' reports can be lost. The pair stays consistent, because the stamp comes from the linked copy. A race that still leaves a mismatch reads in S2 as "out of step";
  - two slots only: a project scan after a Scan Now keeps that Scan Now and drops the report before it (Technical Notes). History is S8's.
- [ ] **README**: a "Where things live" row for `previous-scan.json`; the `last-scan.json` row says that `ddc clean` and `ddc clean --items` both edit it after cleaning.
- [ ] **Tests that list the data files**: the load harness's `owner_state` (`crates/ddc-loadgen/src/run.rs`) watches `previous-scan.json` too, so a load run still proves it never touched the owner's files.
- [ ] **Goldens** (`crates/ddc-cli/tests/golden_cli.rs`): `written_at` and `previous_written_at` normalised as `generated_at` is; `rules_in_force` normalised too (to its count, or left out), or every new rule moves every scan golden. Then `make golden-record`, with a diff that holds only the new fields.
- [ ] **The disk cost**: the size of one saved report on the load fixture home, measured once with `ddc-loadgen` and written into the Final Summary (a fixture, not an owner machine).
- [ ] **The research note follows the story**: its amendment of 2026-11-03 (the rotation rule as settled here).

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

- [ ] Test (store, temp data dir): a `scan`, a `background` and a `clean-recheck` save over a whole, uncancelled report of the default roots each move that report to `previous-scan.json` byte for byte; the new report's `previous_written_at` equals the moved report's `written_at`; no temp file is left.
- [ ] Test: a `clean`, `clean-items` (its pruned and its full-walk case) or `disk-free` save leaves `previous-scan.json` byte-identical and carries `previous_written_at` forward from the report it replaces.
- [ ] Test: a `scan` save keeps nothing, and `previous-scan.json` stays as it was, when the outgoing report is `cancelled`, of other roots or another home, or has a head that cannot be read (then the new report has no `previous_written_at`). The incoming report's roots do not change the decision: a `ddc scan <project>` over a home report keeps the home report.
- [ ] Test (CLI): a home scan A, a home scan B, then two `ddc scan <project>` runs: `previous-scan.json` is B (the first project scan kept it; the second found a project report and kept nothing), and the last report's `previous_written_at` is B's `written_at`.
- [ ] Test: with a folder in the place of `previous-scan.json`, a `scan` save still writes `last-scan.json`, prints a `ddc: warning:` that names `previous-scan.json`, exits 0, carries `previous_written_at` forward and leaves no temp file.
- [ ] Test: each of the six saves writes its own `written_by` and a `written_at` to the second; an edit keeps `generated_at`; `ddc scan --json` prints the same `written_by`, `written_at` and `previous_written_at` as the saved file.
- [ ] Test: `rules_in_force` equals the sorted ids of `ddc rules --json` for the platform, in a full scan and in a pruned re-check's report (`recheck::pruned`); a rule in `disabled_rules` is still listed; an edit keeps the edited report's list.
- [ ] Test (CLI, fixture home): `ddc clean --yes` removes the selected `safe` items; afterwards `last-scan.json` lacks them, its totals drop by their bytes, the touched volume has `read_at`, and `written_by` is `clean`; `ddc space --json` then gives `baseline.source: clean`. `ddc clean --dry-run` and a declined prompt save the re-check with `written_by: clean-recheck`, which keeps the report before it by the rule.
- [ ] Test: `--data-dir` and `DDC_DATA_DIR` put `previous-scan.json` in that folder.
- [ ] Test: the committed `docs/scan.schema.json` and `docs/space.schema.json` equal the generated ones (`committed_*_schema_is_current`), and a report with the new fields validates.
- [ ] The app needs no change: DDCKit's `ScanReport` decodes by explicit coding keys and ignores the new ones; `make ddckit-test` passes against the regenerated schemas.
- [ ] `make gate-fast` passes, and the Linux and Windows legs; a test that cannot run on one of them is listed in `.config/native-skips/`.

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

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
- `write_last_scan` returns whether it kept a report (`kept: Option<written_at>`) beside the `&mut` stamp; `ddc changes`'s block after `ddc scan` uses it, since `previous_written_at` is also carried forward.
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

*Last updated: 2026-11-03*

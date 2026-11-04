---
type: story
id: "story-ddc-changes-what-grew-between-the-last-two-scans"
epic: "DDC-001"
title: "ddc changes: what grew between the last two scans"
status: in-progress
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-03
updated: 2026-11-04
external_refs: {}
tags: [cli, growth, agents]
code_refs: []
specs: []
started_at: "2026-11-04T12:28:25.671Z"
closed_at: null
plan_updated_at: "2026-11-04T12:28:25.671Z"
---

# ddc changes: what grew between the last two scans

| Field | Value |
|---|---|
| **Epic** | [DDC-001](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

On 2026-11-02 the owner's disk lost about 200 GB in a day, mostly to cargo `target/` folders that coding agents built in worktrees. Since 0.28 `ddc space` says that used space grew. Nothing yet says where it went. The new folders show up in the next scan as plain rows, and the new rebuildable ones are not even selected (they are in active projects), so no number the person looks at moves.

This is S2 of the research [[research/what-grew-since-the-last-scan-a-ux-concept-for-the-macos-app|What grew since the last scan: a UX concept for the macOS app]]. A new command, `ddc changes [--json] [--data-dir DIR]`, compares the two reports that the story "Keep the previous scan report, and say who wrote each" (S1) keeps, and says where the growth went:

- **on the disk:** "198 GiB more in use on the startup disk since yesterday 10:04", from the two reports' free space;
- **by group:** "Build outputs +186 GiB, 14 new, 9 in Claude Code worktrees", with the Review groups of `ddc rules --json`;
- **the rest, named for what it is:** "Elsewhere in your home folder" for growth that is no finding, and "In folders ddc did not read, or outside your home folder" with those folders listed;
- **your own cleans apart:** "You cleaned 40 GiB since then; 12 GiB of it is in the Trash", from the action journal, never counted as shrinking or growth.

It compares only what both scans read and only rules both scans had, so a folder read for the first time, or a rule new in this version, never reads as "+30 GiB new". It reads two files and the journal. It walks nothing, takes no lock, writes nothing and cleans nothing.

Who uses it: a person in a terminal (`ddc scan` also prints a short block of it after its summary); the menu bar app, which shows it in a later story (S4, the "Grew since …" entry, not here); and a coding agent, which may show the person what its builds made, within the bound of ADR "Agent footprint: worktrees, harness data and bounded consent for agents" §7.

## Decomposition

- [ ] **Core: load the two reports** (a new `crates/ddc-core/src/changes.rs`). Both files are read the same way. A report of our `SCHEMA_VERSION` decodes strictly. A report of another version is decoded with today's types: bumps 3 to 7 only added enum variants and bump 8 (0.30.0) changed `dev` for btrfs and ZFS findings, which a report from before S1 never reaches since it stops at `predates-fields`; so a report of version 2 or later written by an older CLI normally decodes; version 1 lacks the fields bump 2 made required (`platform`, `capabilities`), and a newer CLI's report may use a variant we do not know. A report that does not decode is `other-schema` (`store::read_last_scan` returns `None` for another version, so the loader reads the bytes itself). Both need `written_at` and `rules_in_force` (S1). The pair belongs together when the current report's `previous_written_at` equals the previous report's `written_at`; if not, both are read once more, then the result is `out-of-step`.
- [ ] **Not compared, with one reason**, exit 0. When several hold, the first in this order is given: `no-report`, `no-previous`, `other-schema` (either side), `predates-fields` (either side lacks `written_at` or `rules_in_force`), `cancelled` (either side), `no-shared-root`, `out-of-step`. Text: "Growth shows after the next scan." for the first four; one sentence that says why for the other three.
- [ ] **Roots compared.** Equal roots compare everything. When one side's root lies inside the other's (a project scan against a home scan), only findings under the inner root are compared, and the output names it ("Compared in ~/Projects only: the earlier scan covered other folders"). No shared root: `no-shared-root`.
- [ ] **What each report did not read.** The union of `not_scanned`, the `skipped` paths and the `inaccessible` paths, each with a kind that picks its label and action: `cloud` (`excluded`, under `Library/CloudStorage` or `Library/Mobile Documents`, the test of `ReviewItems.cloudSkipped`); `skip` (at or under `not_scanned`); `prompting` (any other `excluded`, the test of `ReviewItems.backgroundSkipped`); `refused` (`inaccessible`); `evicted` (`dataless`); `other-disk` (`other-filesystem`, `automount`); `snapshot` (Linux); `changed`. A `holds-subvolume` path is **not** in the set: the walk entered and counted it (`SkipReason` in `crates/ddc-core/src/scan/mod.rs`).
- [ ] **Matching.** A finding's identity is `(path, rule)`; cargo deletes and re-creates `target/`, so `(dev, ino)` is not the key. A `gone` and a `new` with the same `(dev, ino)` and rule are one `moved` row with `moved_from` (a renamed worktree). A finding of either side at or under a not-read path of the *other* side is `not-compared`, whichever way it went. So is a finding whose rule is not in the baseline's `rules_in_force`. `ignored` findings compare like the rest and carry `ignored` from the current report.
- [ ] **Harness.** A finding of a harness rule has its own `origin.harness`. An artifact inside a linked worktree has no `origin` (`rules/classify.rs` sets it only on the worktree's own finding); it has `project.worktree_of`, and its harness is that of the `git.worktree` finding whose path is its `project.root`, in the same report. No such finding: no harness.
- [ ] **Your cleans, from the journal.** A lenient reader of `<log-dir>/actions.jsonl` (today it is written only, `exec/journal.rs`). It takes `item` lines (or version 1 lines with no `kind`) with outcome `deleted`, `trashed` or `native`, dated at or after the baseline's `written_at`, and `area-action` lines of a `brew-cleanup` that ran. A line names a baseline finding when:
  - its `path`, `dev` and `ino` are the finding's; or
  - it is a picked entry: its `path` is a direct child of a baseline finding with `contents` (a user-data folder, a session folder); or
  - it is a session's companion: its `entry` is such a child (the line then counts against that folder's finding);
  - a `brew-cleanup` line names the `cache.homebrew` finding.

  A named finding that is gone is `cleaned` whole. A named finding that shrank has `cleaned` = the smaller of its shrink and the bytes of the lines that name it; it is `cleaned` when that covers the whole shrink, else `shrank` with the rest. The trashed part is the smaller of the credit and the bytes of its `trashed` lines; the deleted part is the remainder. `cleaned_since {deleted, trashed}` sums these credits only, so a line that names no baseline finding counts nowhere. Entry and companion paths never appear in the output; their folder's row does. The `trash` finding's growth up to `cleaned_since.trashed` is `trashed_in`, never growth; only the part above it is.
- [ ] **The three levels.**
  - `disk`: the volume in both reports whose `roots` hold the home folder (else the compared roots' volume); in use = `total_bytes` − `free_bytes`; each side's time is the volume's `read_at`, else the report's `generated_at`. Null when either side lacks that volume.
  - `home`: compared only when both reports have the same home, roots and `full_disk_access`, neither list was cut (`skipped_total` above `skipped.len()`, or `inaccessible` at its 10 000 cap), and the same not-read paths apart from `changed` ones. Otherwise `compared: false` with the reasons. When compared: `delta` from `totals.scanned_bytes`, `ignored_delta`, and `elsewhere`, the remainder after the groups, the ignored findings and your cleans: growth in folders that are no finding, plus findings not compared.
  - `rest`: what the disk delta leaves after the home (or, when the home does not compare, after the findings and your cleans), with `not_read` listing the *current* report's not-read paths and their kinds.
- [ ] **Groups** by the labels of `rules::listing::GROUPS` (the Review groups `ddc rules --json` gives): `{group, delta, new, grew, shrank, gone, cleaned, by_harness: [{harness, delta, count}]}`. `delta` is the net change with your cleans and the Trash's intake left out. Ignored findings are left out of the groups and summed in `home.ignored_delta`.
- [ ] **Rows.** JSON lists every finding that changed: `{path, rule, kind: new | grew | shrank | gone | cleaned | moved | not-compared, before, now, delta, cleaned, trashed_in?, moved_from?, harness?, ignored}`. The text keeps a change of at least 100 MiB or one of the 20 biggest, folds the rest into "N smaller changes", and writes "≈" before a size with `shared_bytes > 0`. Sizes are `bytes` (allocated).
- [ ] **Schema** `docs/changes.schema.json`, `CHANGES_SCHEMA_VERSION` 1: `{schema_version, status: compared | not-compared, reason, baseline, current, compared_roots, disk, home, rest, cleaned_since, groups, items}`. When `status` is `not-compared`: `reason` is set; `baseline` and `current` hold `{written_at, written_by, generated_at}` for each report that decoded, else null; `disk`, `home`, `rest` and `cleaned_since` are null; `compared_roots`, `groups` and `items` are empty. When `compared`, `reason` is null and the rest is filled. `committed_changes_schema_is_current` with `DDC_UPDATE_SCHEMA`.
- [ ] **CLI** (`crates/ddc-cli/src/cmd/changes.rs`, `cli.rs`): `Command::Changes(ChangesArgs {json, data_dir})` in `prints_json()`; in `gate.rs`'s capability match with no capability, as `Space`. Text: the disk sentence, one line per group that grew, "Elsewhere in your home folder", "Not read, or outside your home folder" with the kinds, the cleans line, then the rows. Sizes with `size::human`, the volume with `notify::volume_name`. The age ("yesterday 10:04", "26 h ago") comes from `cmd/space.rs`'s private `age()`, moved to a shared module so `space`, `changes` and `scan` say it the same way. Exit codes: 0 when it compared or said why not; 1 when a report or the journal exists but cannot be opened or read (any error but "not found"), with a `ddc:` line and no JSON; 2 on a usage error; 69 on an unsupported platform; never 75, since it takes no lock. A journal line that does not parse is skipped, not an error.
- [ ] **`ddc scan`'s block** (`cmd/scan.rs`): after the summary, never with `--json` or `--background`. It shows only when this save kept the previous report: `store::write_last_scan` returns the stamp of S1, and a kept report is what makes the block worth showing. The current report is already in memory; the block parses `previous-scan.json` and the journal once more, which is the cost of a second report's parse, measured with the load fixture below. It prints the disk sentence, the top groups, the rest, and "`ddc changes` lists every row."
- [ ] **Tests that list the commands**: a case in `crates/ddc-cli/tests/cases.json` ("changes --json in a fresh home has nothing to compare"), `changes` in its `data_dir_commands`, and both forms in `unported_cli.rs`. A structure test, as `space_reaches_no_cleanup_lock_or_report_write`: `cmd/changes.rs` and `changes.rs` mention no executor, plan, `Consent`, `remove_`, `Command::new`, `try_lock`, `write_last_scan`, `write_last_disk`, `write_atomic` or `Notifier`.
- [ ] **ADR "Menu bar app: Swift shell over the ddc CLI"**: a new §6 item 15, `ddc changes` (its rules, schema, reason order and exit codes; reads two reports and the journal; no lock; writes nothing), after S1's item 14.
- [ ] **ADR "Agent footprint: worktrees, harness data and bounded consent for agents"** §7: one line that `ddc changes --json` is a read-only command the guidance may run; its bound does not change (Technical Notes).
- [ ] **README**: a "`ddc changes`" section in the command-line guide (`readme_cli.rs` checks it against `--help`); "For coding agents" names it among the read-only commands.
- [ ] **Agent guidance** (`agents/claude-code/ddc/SKILL.md`, `agents/AGENTS.snippet.md`): `ddc changes --json` under "What to run", with the §7 bound in plain words. The `structure.rs` guidance tests stay green. If the story "Agent skill v0: ask ddc before guessing about the disk" is still open, the line goes there instead.
- [ ] **The research note follows the story**: its amendment of 2026-11-03.

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

From the planner's placement plan (2026-11-03), checked against `main` at 0.32.0, where S1 is done. Two pull requests.

**S2a: the comparison and `ddc changes`.**
1. `crates/ddc-core/src/changes/`, a folder rather than one file: `mod.rs` (the types, `CHANGES_SCHEMA_VERSION` 1, a pure `compare(baseline, current, cleans, catalog)`), `load.rs` (both files read as bytes, decoded whatever their `schema_version`; the pair check with one re-read; every reason computed, the first in the story's order taken), `journal.rs` (a lenient reader of `actions.jsonl`; `exec/journal.rs` and its coverage floor stay untouched), `not_read.rs` (the kinds; the cloud paths come from the `user.cloud` rule, not from strings), `matching.rs` (`(path, rule)`, then `moved` by `(dev, ino)`, then `not-compared`, then the cleans). The groups come from `rules::listing`, the harness from `origin.harness` or the `git.worktree` finding at `project.root`.
2. `docs/changes.schema.json` with its drift test, in a new `crates/ddc-core/tests/changes.rs` that also holds the core ACs on hand-built reports and journal lines: the sums test first, with its expected numbers written by hand. These tests scan nothing, so they run on all three systems.
3. CLI: `Command::Changes` in `cli.rs` and `gate.rs` (no capability, as `space`), `cmd/changes.rs` shaped like `cmd/space.rs` (never a lock; exit 1 with a `ddc:` line and no JSON). `age()` moves from `cmd/space.rs` to `render.rs` with its tests.
4. `crates/ddc-cli/tests/changes_cli.rs` (AC 6, 10, 11's folding), `cases.json` and `cases_cli.rs`, `unported_cli.rs`, a `structure.rs` test that the command reaches no executor, lock or report write, and the Windows skip names.
5. README `### ddc changes`; ADR "Menu bar app" §6 item 15. Minor bump.

**S2b: the `ddc scan` block and the guidance.** `cmd/scan.rs` prints the short form after the summary when this save kept a report (`Kept::Kept`), never with `--json` or `--background`; the CLI test of AC 11's block half; README "For coding agents"; the guidance line through the story "Agent skill v0: ask ddc before guessing about the disk" while it is open; ADR "Agent footprint" §7. Patch or minor bump by what it adds.

Decided here: a `brew-cleanup` journal line credits `bytes_before` minus `bytes_after` when both are present, else the whole shrink of the `cache.homebrew` finding. Times in the Final Summary are ratios to a `ddc scan` of the same home only (Technical Notes).

## Acceptance Criteria

- [ ] Test (core, two fixture reports): a finding under a folder the baseline did not read (`Downloads`, skipped as `excluded` by a scheduled scan without Full Disk Access) and read now is `not-compared`, never `new`; the reverse is `not-compared`, never `gone`; a user skip removed since is `not-compared`; a path skipped as `holds-subvolume` changes nothing in the matching.
- [ ] Test: a finding whose rule is missing from the baseline's `rules_in_force` is `not-compared`; a finding that was `ignored` in the baseline (a type switched on since) compares as usual.
- [ ] Test: a `target/` deleted and rebuilt at the same path (new `ino`) is one `grew` or `shrank` row, not `gone` plus `new`; a worktree folder renamed is one `moved` row with `moved_from` and only its size change.
- [ ] Test (fixture reports and a journal file): an item deleted through `clean --items` between the two saves is `cleaned`, not `gone`; a trashed one is `cleaned`, and the `trash` finding's growth up to that amount is `trashed_in`; a folder that went with no journal line (removed in Finder) is `gone`; a `brew-cleanup` area line makes the `cache.homebrew` shrink `cleaned`; a line with the same path but another `ino` explains nothing and adds nothing to `cleaned_since`.
- [ ] Test: a picked `~/Downloads` entry trashed through `--items` makes the `user.downloads` row's shrink `cleaned` up to the line's bytes; a session's transcript and its companions trashed together count against their session folder's finding; neither the entry's path nor a companion's appears among the rows.
- [ ] Test (CLI, S1's flow): scan, `clean --items` of one `target/`, rebuild it, scan: the row is `new`, not `shrank`, and the clean does not appear, since the baseline is the edited report.
- [ ] Test: a scheduled scan without Full Disk Access against a scan with it gives `home.compared: false` with `different-access`, and group deltas only; a difference only in `changed` paths still compares the home; a cut `skipped` list does not.
- [ ] Test (sums): two hand-built reports and a journal whose expected result is written in the test, not computed by the code under test: one group with a new 10 GiB finding and a 2 GiB growth; 3 GiB deleted and 1 GiB trashed by journaled cleans; the `trash` finding grown by 1.5 GiB; a 4 GiB growth in no finding; an ignored finding grown by 0.5 GiB; free space down by 20 GiB. Expected: the group's `delta` 12 GiB with `new: 1, grew: 1`; `cleaned_since` 3 GiB deleted and 1 GiB trashed; the Trash's `trashed_in` 1 GiB and its group `delta` 0.5 GiB; `ignored_delta` 0.5 GiB; `home.delta` 14 GiB; `elsewhere` 4 GiB; `rest.delta` 6 GiB.
- [ ] Test: a `rust.target` finding inside a Claude Code worktree counts in its group's `by_harness` as `claude-code`, through the `git.worktree` finding at its `project.root`.
- [ ] Test: each reason (no report, no previous, a report on either side that does not decode, one without S1's fields, a cancelled side, no shared root, a `previous_written_at` that still does not match after the re-read) gives `status: not-compared`, that reason, the JSON shape of the schema item (nulls and empty lists), and exit 0; with two reasons at once the earlier in the order wins; a previous report of an older `schema_version` that decodes is compared.
- [ ] Test: a project scan as the current report against a home baseline compares only under the project's root and says so.
- [ ] Test (CLI): `ddc changes --json` validates against the committed `docs/changes.schema.json`, which equals the generated one; it runs while another process holds the exclusive lock; it leaves every file in the data and log folders as it was (bytes and mtime); `--data-dir` and `DDC_DATA_DIR` are honoured; an unreadable `last-scan.json` (no read permission) exits 1 with a `ddc:` line; a journal with a broken line still gives the other lines' cleans.
- [ ] Test (CLI): the text output folds rows under 100 MiB beyond the 20 biggest into "N smaller changes"; `ddc scan` prints its block after the second scan of a fixture home and not after the first, and never with `--json`. The goldens of a fresh data folder do not change.
- [ ] `make gate-fast` passes, and the Linux and Windows legs; a test that cannot run on one of them is listed in `.config/native-skips/`.

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- **Group names.** The research's examples ("Rust build folders", "Node packages", "Xcode DerivedData", "Docker", "Package caches") are not the code's groups. `rust.target` sits in "Build outputs" with Maven, Gradle, CMake and CocoaPods outputs; the others are "node_modules", "Xcode builds", "VM and container images" and "Package downloads" (`crates/ddc-core/src/rules/listing.rs`). `ddc changes` uses the code's labels, the same words the app's Review window shows. If "Rust build folders" should be said, that is a per-rule line inside a group, decided at the plan gate or in S4.
- **Harness.** `origin` is set only on a harness rule's findings and on a worktree's own finding (`rules/classify.rs`, `worktree_finding`). An artifact in a worktree gets `project.worktree_of` instead, so "9 in Claude Code worktrees" comes from the worktree finding at its `project.root`. An artifact in an orphan worktree has no `worktree_of` and no harness.
- **The journal** lives in the log folder (`~/Library/Logs/DeepDriveCleaner/actions.jsonl`, or `DIR/logs` under `--data-dir`), not the data folder. Its times are to the second, like `written_at`, so the window is inclusive and the `(path, dev, ino)` match (or the entry's folder) keeps a clean that starts in the second a scan saved from being misread. It grows without bound and is read whole; that is fine at today's sizes and accepted for now. A reverse read that stops at the baseline's time is the fix if it ever is not.
- **What `home.compared` ignores.** A folder that changed while the walk listed it (`changed`) is common exactly when builds run, the owner's case. Requiring the same `changed` paths on both sides would leave the home uncompared most of the time. Findings under those paths are still `not-compared`; only the home total ignores the difference. The research asked for the same not-read set; this narrows it on purpose.
- **A false `moved`.** On ext4 (and other file systems that reuse inode numbers quickly), a folder removed between the scans can leave its `(dev, ino)` to a new folder of the same rule elsewhere, which then reads as `moved`. Accepted: the row names both paths and counts only the size change, so the totals stay right.
- **Two slots.** S1 keeps two reports. After an agent's project scan that followed a Scan Now, the baseline is that Scan Now; the output says the baseline's age, and "since yesterday" waits for S8.
- **Not in `ddc changes`:** the count of local snapshots (the app has it from `ddc disk`; S4 adds the snapshots sentence); folder sizes for growth that is no finding (S3; until then one "Elsewhere" line); "since yesterday or last week" (S8); a growth notification (S6). The app's band and Review entry are S4.
- **Older reports.** Since S1 every report carries `rules_in_force`, so a release that adds a rule does not show its findings as new: they are `not-compared` until two reports both have it. A release that bumps `SCHEMA_VERSION` by adding enum variants still compares, since the older report decodes with today's types.
- **Agents.** `ddc changes --json` lists the whole home folder. The guidance tells an agent to show only what the person asked about, and never to remove a `rebuildable` item itself, its own worktrees' `target/` folders included; it hands the person the `ddc clean --items` command or the Review window (ADR "Agent footprint", §7). For one project, `ddc agent footprint` stays the scoped command once it ships.
- **Performance.** Two full reports parsed once each and a hash join over their findings; no walk. Measure `ddc changes` and the `ddc scan` block's extra parse once on two reports of the load fixture home and write into the Final Summary only the ratio to a `ddc scan` of the same home, no run times (the project keeps owner-machine timings out of the public repos).
- **Privacy.** The output lists the same paths as `ddc scan`. Nothing is written or sent.
- **Linux and Windows.** The command is platform-neutral and needs no capability. The `cloud` kind exists only where the catalog has cloud folders (macOS today); `snapshot` and `other-disk` come from Linux. Their apps follow later.
- Harness: a `git.worktree-orphan` finding at the artifact's `project.root` counts too (its `origin.harness` is set), so agents' worktrees whose main checkout is gone still read "in Claude Code worktrees".
- `ddc scan`'s block after a save that kept a report but cannot compare (after an upgrade, `predates-fields`) prints the one-line reason, nothing more; an AC beside the block's others covers it.
- The unreadable-report AC uses a folder in place of `last-scan.json`, which fails to read on every platform without a native skip, not a missing read permission.

## Dependencies

- [[epics/DDC-001/stories/story-keep-the-previous-scan-report-and-say-who-wrote-each|Keep the previous scan report, and say who wrote each]]: the two reports and their fields; this story cannot start before it
- [[research/what-grew-since-the-last-scan-a-ux-concept-for-the-macos-app|What grew since the last scan: a UX concept for the macOS app]]: the concept; this is its S2
- [[epics/DDC-002/stories/story-popover-say-when-used-space-grew-since-the-last-scan|Popover: say when used space grew since the last scan]]: `Volume.read_at` and the wording of `ddc space`, reused here
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]: §6 item 15 added
- [[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]]: §7, what an agent may do with the output
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]: the action journal it reads
- [[epics/DDC-003/stories/story-agent-skill-v0-ask-ddc-before-guessing-about-the-disk|Agent skill v0: ask ddc before guessing about the disk]]: the guidance that gains the command

## Attachments

-

---

*Last updated: 2026-11-04*

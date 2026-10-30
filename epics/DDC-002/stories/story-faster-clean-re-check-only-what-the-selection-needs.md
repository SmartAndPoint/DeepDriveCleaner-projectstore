---
type: story
id: "story-faster-clean-re-check-only-what-the-selection-needs"
epic: "DDC-002"
title: "Faster clean: re-check only what the selection needs"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-21
updated: 2026-10-30
external_refs: {}
tags: [cli, app, safety, performance]
code_refs: ["README.md", "app/DeepDriveCleaner/AppDelegate.swift", "app/Packages/DDCKit/Sources/DDCKit/CleanReport.swift", "app/Packages/DDCKit/Sources/DDCKit/CleanRun.swift", "app/Packages/DDCKit/Sources/DDCKit/CleanTimings.swift", "app/Packages/DDCKit/Sources/DDCKit/PopoverModel.swift", "app/Packages/DDCKit/Sources/DDCKit/ReviewModel.swift", "app/Packages/DDCKit/Tests/DDCKitTests/CleanRunTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/CleanTimingsTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ContractTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/FakeCLI.swift", "app/Packages/DDCKit/Tests/DDCKitTests/PopoverModelTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ReviewSupport.swift", "app/Packages/DDCKit/Tests/DDCKitTests/Fixtures/clean-report.json", "crates/ddc-cli/src/cmd/clean_items.rs", "crates/ddc-cli/tests/clean_cli.rs", "crates/ddc-cli/tests/clean_items_cli.rs", "crates/ddc-core/src/exec/report.rs", "crates/ddc-core/src/progress.rs", "crates/ddc-core/src/report/after_clean.rs", "crates/ddc-core/src/report/mod.rs", "crates/ddc-core/src/report/recheck.rs", "crates/ddc-core/src/rules/catalog.rs", "crates/ddc-core/src/rules/mod.rs", "crates/ddc-core/src/rules/toolchain.rs", "crates/ddc-core/src/scan/mod.rs", "crates/ddc-core/src/scan/walker.rs", "crates/ddc-core/tests/after_clean.rs", "crates/ddc-core/tests/recheck.rs", "crates/ddc-core/tests/common/fixtures.rs", "docs/clean.schema.json", "docs/progress.schema.json"]
specs: []
started_at: "2026-10-25T09:17:22.397Z"
closed_at: "2026-10-30T06:45:42.172Z"
plan_updated_at: "2026-10-25T09:17:22.397Z"
---

# Faster clean: re-check only what the selection needs

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

A clean from the Review window waits about half a minute before it touches anything, even for a few small caches. The owner measured it on the reference machine on 2026-10-21, with the 0.2.2 build, cleaning two items of "Package caches" (51.7 MiB: one cache deleted, one tool's own clean command):

| Moment (app log, journal) | Time |
|---|---|
| Clean pressed in the confirmation sheet | 18:46:28.6 |
| Both actions recorded in the journal | 18:46:58 |
| The clean process ended | 18:46:58.9 |
| The app's own scan after the clean | 18:46:58.9 → 18:47:36 |

The actions themselves took under a second; everything before the first action took about 29 s. That time is attributed to the full rescan that `ddc clean --items` runs first, because a full scan of the same home took 37 s right after. It is an inference, not a measurement. Other steps also run before the first action, untimed:
- the process snapshot with every process's open files (added in 0.2.1);
- `tmutil listlocalsnapshots`;
- the keep-marker walk of the chosen items.

Decomposition step 1 times them before anything changes. After the clean, the app scans the whole home again to refresh Review and the popover, another 37 s of disk work.

The full rescan is a recorded decision, not an accident:
- **The safety ADR "Cleanup safety model: risk classes, plan and confirmation".** §5 says "`ddc clean` **always rescans**". Its Consequences accept "The rescan in `clean` costs time".
- **The menu bar ADR "Menu bar app: Swift shell over the ddc CLI".** §6 item 1 says "Matching. It rescans, as every clean does". Its Open Question was decided at the plan gate of story "CLI: clean an explicit item list" (2026-10-18): the full rescan stays, because only a whole-tree walk sees the toolchain pins (`toolchain::dropped` needs every pin file and sibling version the walk found) and the ancestor claims (a rule that would now claim a directory above an item). The same gate rejected a scoped walk because it "would need a second classification path".

The owner chose two changes on 2026-10-21 ("1 и 2"):

1. **Re-check only what the selection needs.** `clean --items` walks the whole home only when its list holds an item whose verdict depends on files anywhere in the home: today the protected toolchain versions (`toolchain.nvm`, `toolchain.pyenv`, `toolchain.rustup`), whose pins can sit in any project.

   For any other list it walks the listed items and checks what lies above and beside them:
   - a rule that now claims an ancestor;
   - a keep marker;
   - an exclude;
   - a lock file or `.git` that changes the item's class or its project root;
   - for an item of a `KnownChildren` rule, its siblings, which decide whether it is a finding at all.

   It uses **one classification path**: the same rule visitor, classification and `plan_listed` as today, run over a walk pruned to those directories. A picked entry of a user-data folder also gets its folder listed with times, so its `EntryIndex` exists as it does after a full walk. Three cases fall back to the full walk without a safety reason: an unreadable ancestor, no readable report, and a report of other roots or another home (the third added 2026-10-25, gate decision 10a). A rule the catalog lacks is refused as `rule-mismatch` without a walk; the scope field is exhaustive, so no rule's scope is unknown.

   The per-item checks before each action stay as they are.
2. **No second scan after the clean.** The CLI leaves `last-scan.json` describing the disk after the clean (section below), and the app runs no scan of its own after a clean that wrote it. After a crash that followed the `cleaning` event, it still rescans, as today.

These options were weighed and set aside for now:
- **Re-checking through the FSEvents history**, rescanning only the directories changed since the scan: the fastest, but it needs the walk's state persisted between runs. It is the most complex option.
- **Deleting in parallel:** in the measurement above deletion was not the bottleneck. It is worth measuring on a very large `node_modules` first.

### `last-scan.json` after a clean

**Which file.** The CLI edits the report on disk, under its exclusive lock, whichever scan wrote it: the one the user reviewed, or a newer one from a scheduled or terminal scan in between. With no report of the current schema, or none of the clean's roots and home (gate decision 10a), it takes the full walk and writes a fresh report, as today.

**Per outcome:**

| Outcome | What happens in the report |
|---|---|
| Deleted | The item leaves, with its bytes, files and directories in every total it counted in. |
| Trashed | The item leaves. When the report holds the `trash` finding (`~/.Trash`) and the item lies on the home volume, its bytes, files and directories move to that finding: the scanned total does not drop and the `review` class grows. Otherwise the item left the scan roots (the Trash of another volume, or a home outside the roots) and is treated as deleted. |
| Run by its tool | The item is measured again and the difference subtracted. |
| Skipped, failed, not run | The item stays as it was. |
| Picked user-data entry | The entry leaves its folder's `contents` (entries, or the rest count and bytes). The folder finding shrinks by the entry's bytes less those of the findings inside it, which the folder never counted. Those findings leave the list without moving any bytes: the entry's bytes hold them. The entry's bytes follow the Trashed rule once. Its files and directories are not known, so the scanned totals keep them until the next full scan (gate decision 6). |
| A finding inside a listed Downloads entry, deleted or trashed | Besides the rule of its outcome, that entry's `bytes` and `findings` shrink, or the folder's `rest_bytes` when the entry is in the rest (gate decision 6). |

Every total of the report has a rule: scanned bytes, files, directories, unowned bytes, per-class count, bytes and selected figures, and the volume's free space (taken from the clean's own `statfs`).

**What stays.** In the pruned case `generated_at` stays the previous report's, so the popover's "last scan" age and `ddc schedule status` do not claim a fresh scan of the whole home. The app's "Cleaned just now" lock then lasts until the next real scan; the cleaned findings are gone from the file anyway. `duration_ms` and `ddc_version` stay the previous report's too. In the full-walk case all three are the rescan's, as for any full scan.

**When it is written.** Once, after the clean, in both the scoped and the full-walk case: today's full walk writes it before the first action, and that moves too. After a cooperative cancel it is written with what was done, and the not-run items stay. A crash or SIGKILL leaves the file from before the clean, and the app rescans as today when it saw the `cleaning` event.

**The signal that acting begins.** (Superseded in part 2026-10-25, see the note below.) The app can no longer read anything from the file's date. The CLI writes the `cleaning` phase event of story "CLI: progress events" to stderr, flushed, under `--progress json`. It writes it after the journal is open and the signal handler installed, immediately before `exec::execute`, so an exit before that point (such as "cannot open the action journal") is "Nothing was touched". Only the phase events (`preparing`, `cleaning`) are brought forward here, without counters, in a first `docs/progress.schema.json` that "CLI: progress events" extends. `--progress json` is accepted by `clean` as a whole, the terminal's included, with phase events only.

The app reads both of its signals from that event:
- the switch from preparing to cleaning;
- after a crash: event seen → "the journal lists what was done"; not seen → "Nothing was touched".

A crash between the event and the first action errs on the safe side: it points to the journal although nothing was done.

**Note, 2026-10-25.** Story "CLI: progress events" comes first. At its plan gate it moved the `cleaning` event into the executor, after the executor's own preparation and immediately before the first action. It also creates `docs/progress.schema.json` with the full event stream. This story uses both as they are; it no longer brings phase events forward.

The app takes the event from the runner's live `stderrLine` callback, never from `Result.stderrTail`, a bounded tail that warnings or "stopping after the current item" can push it out of. The signal does not survive an app relaunch during a clean. That is acceptable: quitting cancels the clean and waits for the CLI (`prepareToQuit` in `CleanRun.swift`), as today, and the app never resumes watching a clean it did not start.

### Scope

- The terminal's `ddc clean`, with or without `--yes`, keeps the full walk and its order.
- Story "Automatic cleaning of download caches after a scheduled scan" uses `clean --yes` with `--group` and assumes a rescan after it. It is re-read when this story lands; its design is not changed here, and it gets a dated note.
- Lock semantics do not change: a clean holds the exclusive lock, a scan the shared one.

### This story waits for ADR amendments before any code

Each amendment is reviewed by the critic. The text that encodes the rescan, all of it to be amended or dated so no contradiction stays behind:

- **The safety ADR:**
  - §5 "always rescans";
  - Consequences, "The rescan in `clean` costs time";
  - the amendment of 2026-10-18 ("Everything else in §5 still applies…: the rescan").
- **The menu bar ADR:**
  - §5, "The rescan after a clean is part of the clean and runs while paused";
  - §6 item 1, and item 2's "The real clean still rescans";
  - Consequences, "Every clean still rescans first: about 40 s";
  - the decided Open Question, with why the pruned walk sees what the full walk sees for the lists it is used for.
- **The epic "DDC-002":** its rows and notes on the rescan.
- **Stories:**
  - "Review and clean window": its decisions on the rescan after a clean;
  - "Menu bar popover": its criteria that the rescan after a clean runs while paused.

## Decomposition

- [x] Time each step before the first action in the app log: the re-check, and the rest up to the `cleaning` event (the keep-marker walk, local snapshots, process snapshot), bracketed. AC 9's manual check reads it (gate decision 7, which moved this measurement after the change). If another step is the cost, it becomes its own story and AC 9 is re-cut. Done: `CleanTimings.swift`, logged by `CleanRun.swift`; the follow-up waits for AC 9.
- [x] Amend the ADRs and the texts listed above; critic. Amendments of 2026-10-24, dated notes of 2026-10-25; critic approved on the third pass, 2026-10-25.
- [x] Core: give every rule a scope, matched exhaustively (`Item`, `Siblings` or `Tree`), so a new cross-tree rule cannot be missed silently; a list holding a `Tree` rule takes the full walk.
- [x] Core: the pruned walk, a visitor wrapper that skips any directory that is not an ancestor of, inside, or a sibling-for-`KnownChildren` of a listed item. It reuses the rule visitor, classification and `plan_listed` unchanged, and builds the `EntryIndex` of a picked entry's folder.
- [x] Equivalence tests over synthetic homes covering every rule in the catalog: the pruned and the full re-check give the same outcome for every listed item, for each change of AC 4.
- [x] CLI: `last-scan.json` edited after the clean, per the section above. The phase events and `docs/progress.schema.json` come from "CLI: progress events" (note of 2026-10-25).
- [x] App:
  - drop the rescan after a clean that wrote the report; keep it after a crash that followed the `cleaning` event;
  - read the phase and "nothing touched" from the event;
  - reload the report when the clean ends (`PopoverModel.refresh` skips it during a clean today);
  - rewrite the crash tests.
- [x] README and release notes.

## Implementation Plan

Planned 2026-10-25 with the planner, after "CLI: progress events" (DDC-002) landed in 0.2.3. No spec covers it. The binding "how" is the safety ADR's amendment of 2026-10-24 (rule scope, which walk to take, why the pruned walk sees what the full walk sees) and the menu bar ADR §5 and §6 items 1 to 3, with the note of 2026-10-25.

### Decisions taken at this gate

1. **A claim on an ancestor stops measuring there.** The walker asks a new `Visitor::prunes(dir)` before it opens any directory, inside claims too, so a claim above a listed item no longer makes the walker measure the claim's whole subtree: a `.ddc-keep` in `~/Code`, the Downloads region, or the folders claimed whole by design (Pictures, Movies, Music, the device backups), whose unpicked entries are not read. The item is refused as the full walk refuses it: under a claimed ancestor, `plan_listed` decides from the claim existing at the ancestor and the item not being its own finding, never from the claim's size. One reason can differ. When a claim appeared above the item after the scan and a keep marker sits in another of that claim's subtrees, the full walk turns the claim into `user.kept` and refuses the item `kept`, while the pruned walk does not read that subtree and refuses it `changed-since-scan`. Both refuse. In the user's own folders a marker locks only the entry that holds it (amendment of 2026-10-20), so a picked entry is decided the same either way. The equivalence tests compare whole plans and expect that one pair. The re-check report's partial sizes never reach `last-scan.json` (decision 6). AC 1 then holds as written. The safety ADR's sentence that a picked entry's folder is "claimed whole by design", so measured whole, gets a dated note.
2. **Pruned directories are their own count.** `WalkOutput.pruned` counts them; they never count in `skipped`, `skipped_total`, `errors`, an owner, an entry or a root. A listed item's `incomplete` still comes only from its own subtree, which is never pruned. No `SkipReason` variant is added, so the scan schema does not change.
3. **Both walks stay.** The pruned re-check refuses an item with a keep marker inside at plan time; the executor's own keep-marker walk stays as the backstop for markers added in the seconds since, and the `walk markers` step of "CLI: progress events" stays. Revisit only if the timings (decision 7) show the marker walk matters.
4. **AC 1 observes the reader.** The re-check takes a `&dyn DirReader`, the seam `scan::walk_with_reader` and the counting reader in `tests/clean_items.rs` already use. A counting visitor would not see reads inside claims. The counting reader forwards `read_with_times`, or picked entries would come back without times and read as `recent`.
5. **A tool-cleaned item is measured again** after `execute`, under the same exclusive lock, by walking that item alone as the scan measures a finding (clones included). Gone: it counts as deleted. No event is written for it: nothing follows `cleaning` but `items` and `item`. So the sheet stays at 100 % while a large tool-cleaned folder (a Go module cache) is walked again; that is accepted.
6. **The edited report.** In the pruned case the CLI edits the report it read under its lock; in the full-walk case, the rescan's report. Per outcome, as the section above says, with these rules and additions:
   - **Whose figures.** The edit subtracts the report's own figures, never the re-check's: a deleted or trashed finding leaves with the `bytes`, `files` and `dirs` the report holds for it; a picked entry listed in `entries` leaves with the report's `bytes` for it, one in the rest with the clean's measured bytes; a tool-cleaned item changes the totals by the report's bytes, files and directories minus the re-measured ones, which may be negative. Every subtraction stops at zero, so an entry that grew since the scan never wraps a total. An item the report does not hold (a newer scan's report, written in between) edits nothing and moves nothing to the `trash` finding. A picked entry not among its folder's `entries` is taken as one of its rest: an entry the plan accepts existed when that report's scan read the folder.
   - a finding inside a listed Downloads entry that is deleted or trashed also shrinks that entry's `bytes` and `findings`, or the folder's `rest_bytes` when the entry is in the rest;
   - a listed finding skipped as `inside-another-item` leaves with the trashed entry that held it: that rule comes before "Skipped stays";
   - a picked entry's folder shrinks by the entry's bytes less those of the findings inside it, which the folder never counted; those findings leave without moving any bytes, and the `trash` finding gains the entry's bytes once;
   - a picked folder's files and directories are not known (neither the clean's result nor the report's entry carries them), so the scanned `files` and `dirs` keep them until the next full scan; a picked file is one file and no directory; its bytes follow the Trashed rule;
   - with no `trash` finding in the report (an empty or never scanned `~/.Trash`), a trashed item counts as deleted, and the Trash reads low until the next full scan;
   - `large_files`, `inaccessible` and `skipped` lines at or under a removed path leave, and `errors_total` and `skipped_total` drop by as many. Both lists hold at most 10 000 lines, so past that the totals stay high until the next full scan;
   - every finding-derived total (per class count, bytes and selected figures, selected and reclaimable bytes, the ignored figures) is recomputed with the scan's own code, and `ignored` and `selected` are recomputed only for findings whose bytes changed (a tool-cleaned item, the `trash` finding, a folder that lost an entry; such a folder can drop under `min_size` and leave Review until the next scan, as a full scan would decide); `unowned_bytes` stays;
   - the free space of each volume the clean measured is its `free_after`, by mount point; the other volumes keep theirs.
   The edited report drifts from the disk in those few figures until the next full scan replaces it.

   *Note of 2026-10-29* ("Review window: after a clean, only skips that can clear stay selected", DDC-002): "Skipped stays" left a stale `(dev, ino)` in the report for an item changed after the scan, so every later clean skipped it again until a full scan. Since then, a listed item the clean skipped or did not reach takes the re-check's own finding at its path. That finding is no partial size: the pruned walk measures a listed item's whole subtree. A skip that says the item is gone from the disk as scanned (`missing`, `changed-since-scan`, `rule-mismatch`, `not-a-directory`, `symlink`) drops the finding when the re-check has none there. The lines under a refreshed item leave until the next full scan. A `failed` item and a picked entry stay as above. "Whose figures" holds for what the clean removed.
7. **Timing step 1 moves to AC 9's check.** The owner asked on 2026-10-25 for the planned work to be finished without waiting for the owner, and the measurements happen only on the reference machine. The evidence already points at the rescan: the same home's full scan took 37 s right after a clean that waited 29 s before its first action, and that clean ran one. So the app logs one timing line per clean from the events it now receives (requested, process started, `preparing`, the re-check's final tick, `cleaning`, and the marker walk, whose ticks come at most 100 ms apart, so its length is known to about 100 ms), timed on the runner's reader thread. AC 9's manual check reads it. If the re-check is fast and the remainder is not, the remainder becomes its own story and AC 9 is re-cut.
8. **The app learns from the clean report that the file was written.** `CleanReport` gains `last_scan_saved`, an optional field, so the clean schema keeps version 2 under the `model.rs` rule. It is written only by `clean --items`; the terminal's `ddc clean` prints no clean report. Absent, it reads as false: an older CLI. The file's date is what this story stops reading.
9. **A failed write** of `last-scan.json` after the clean prints `ddc: warning: cannot save the scan report: …`, reports `last_scan_saved: false`, and exits as usual; the app then rescans. It rescans too when the report says saved but the app cannot read the file. A signal during the re-check ends the CLI before any write (its handler is installed after the re-check), so the file from before stays; the full walk's report is no longer written before the clean.
10. **An unreadable ancestor** falls back to the full walk inside the CLI, with a fresh counter and the previous report's file count as its estimate; the app's reducer already takes the larger counter.
10a. **A report of other roots** takes the full walk too, for no safety reason. `ddc scan ~/Code` writes a report of `~/Code` alone; editing it would leave the app showing that report after a clean of the whole home. So a report whose `home` or `scan_roots` differ from the clean's counts as no report, and so does one that cannot be read. The roots are compared as the walker normalizes them: canonical, sorted, nested roots dropped, so their order does not matter. The full walk then has no estimate: the other report's file count says nothing about the clean's roots. The safety ADR's list of full-walk cases and menu bar ADR §6 item 1 get a dated note.
11. **One release, one change to `clean_items.rs`.** All parts land before the next release, 0.2.4. The pruned re-check and the report edit change `clean_items.rs` together: today's write of the re-check report before the clean goes in the same change, so no build ever saves a pruned walk's partial report.

### Route

**Part 0: the timing line** (decision 7). `CleanTimings.swift` in DDCKit, a pure value fed with the events and the instants they arrived at; `CleanRun.swift` stamps them in the `stderrLine` and `onStart` closures and logs `clean timings: …` when the run ends.

**Part 1: rule scope and the pruned re-check.**
- `rules/mod.rs`: `pub enum Scope { Item, Siblings, Tree }` and `Rule.scope`; `catalog.rs`: `Item` by default, `Tree` on `toolchain.nvm`, `toolchain.pyenv` and `toolchain.rustup`, `Siblings` on `ide.jetbrains-old-data`. A unit test ties the `Tree` and `Siblings` sets to `toolchain::dropped`, and a catalog test requires every `KnownChildren` rule to be `Siblings` or `Tree`.
- `scan/mod.rs` and `scan/walker.rs`: the `prunes` hook (default false), checked at the top of `visit()`, and the `pruned` counter (decision 2).
- `report/mod.rs`: the scan pipeline moves into one internal function that takes a reader and an optional reach; `scan`, `scan_with` and `scan_with_entries` keep their signatures and readers. The sort, the finding-derived totals and the ignored rule become shared helpers for Part 2.
- `report/recheck.rs` (new): `walk_for(list, catalog, previous)` matches `rule.scope` exhaustively; `Reach` holds the listed paths, their ancestors and the parents of `Siblings` items in hash sets, O(depth) per directory; a `Pruned` visitor wraps the rule visitor and delegates every hook, so classification, `contents::attach` and `plan_listed` are unchanged; `same_roots(report, env, opts)` filters the report read, so a report of other roots or another home reaches `walk_for` as none (decision 10a); `pruned(…)` returns the report and the entry index, or `UnreadableAncestor`; `expected_files` sums the listed findings' files and, for a `Siblings` item, its sibling findings'.
- `progress.rs`: the doc of `files_expected` for `recheck`; `docs/progress.schema.json` regenerated, version unchanged.
- `cmd/clean_items.rs`, landing with Part 2 in one change (decision 11): read the report once under the lock, choose the walk, run it under `walk recheck`, fall back on `UnreadableAncestor` (decision 10).

**Part 2: `last-scan.json` after the clean.**
- `report/after_clean.rs` (new): `remeasure` (decision 5) and `edit` (decision 6), pure over the report.
- `exec/report.rs`: `last_scan_saved`; `docs/clean.schema.json` regenerated.
- `cmd/clean_items.rs`: no write before the clean; after `execute`, re-measure the tool-cleaned items, edit, write, then print the clean report (decisions 8, 9). The same after a cooperative cancel.
- `README.md`: the `clean --items` paragraph and the `last-scan.json` row.

**Part 3: the app.**
- `CleanReport.swift`: `lastScanSaved`.
- `PopoverModel.swift`: `reloadReport()`, which loads the file also during a clean.
- `ReviewModel.swift`: `reloadReport` in place of `pollInterval`; `AppDelegate.swift` wires it.
- `CleanRun.swift`: the modification-date watcher and the baseline go; the crash rule is the `cleaning` event alone; a saved report is reloaded with no scan; otherwise the rescan runs as today.
- Tests: `FakeCLI` writes its report after `clean-go`; the flow, cancel and crash tests are rewritten; the copy of step 1 may change.
- `docs/releases/0.2.4.md`.

**Unchanged.** The terminal's `ddc clean` keeps its full rescan before its plan (AC 8). The story "Automatic cleaning of download caches after a scheduled scan" uses that path, so its assumption of a rescan still holds; it gets a dated note.

**Tests.** AC 1, 2 and 5 in `tests/recheck.rs` with the counting reader; AC 4 and 5 iterate the catalog through a fixture table lifted from `tests/rules.rs` into `tests/common/`, keeping its "no fixture for rule" failure, one test per change of AC 4, each asserting `plan_listed` of the pruned and of the full re-check are equal; AC 6 in `tests/after_clean.rs` and `tests/user_data.rs`, each also checking the totals against the shared helper; AC 3, 6 and 8 at the CLI; AC 7 in `CleanRunTests`, `PopoverModelTests` and the three CLI tests of "CLI: progress events" that already cover the event.

**Pitfalls.** The schema staleness tests (progress and clean schemas change; `model.rs` doc comments must not); `ContractTests.cleanHasOneOwner` (one `.clean` token in `CleanRun.swift`); `structure.rs`; the README test of option names; `disallowed_methods` in tests (a file disappearing is a rename to a holding directory); the equality oracle needs the `Pruned` visitor to delegate `wants_private`; a `Tree` item never takes the pruned walk.

## Acceptance Criteria

- [x] **AC 1.** With no `Tree` rule in the list, `clean --items` opens no directory outside the listed items, their ancestors, and the siblings of a `KnownChildren` item with their subtrees. A test asserts the opened set on a synthetic home with a large unrelated tree. An unreadable ancestor falls back to the full walk (test). — evidence: `crates/ddc-core/tests/recheck.rs` `the_pruned_recheck_opens_only_the_items_their_ancestors_and_a_siblings_family` (counting reader), `an_unreadable_ancestor_sends_the_recheck_to_the_full_walk`; `crates/ddc-cli/tests/clean_items_cli.rs` `a_pruned_clean_edits_the_report_it_read`, `an_unreadable_ancestor_takes_the_full_walk`; reviewer 2026-10-25
- [x] **AC 2.** With a `toolchain.nvm`, `toolchain.pyenv` or `toolchain.rustup` item in the list, it walks the whole tree, and a pin file added after the scan protects that version (test). — evidence: `crates/ddc-core/tests/recheck.rs` `a_toolchain_item_takes_the_full_walk_and_a_new_pin_protects_its_version` (nvm, pyenv, rustup)
- [x] **AC 3.** A list with no report of the current schema on disk, a report that cannot be read, or one of other scan roots or another home (gate decision 10a), takes the full walk and writes a fresh report (test). — evidence: `crates/ddc-cli/tests/clean_items_cli.rs` `without_a_report_of_these_roots_the_clean_walks_the_whole_home` (none, unreadable, other roots); `crates/ddc-core/tests/recheck.rs` `only_a_report_of_the_same_roots_and_home_is_edited`, `no_report_to_edit_takes_the_full_walk`
- [x] **AC 4.** After the scan, each of these changes gives the listed item the same outcome as the full rescan gives today (tests):
  - a keep marker in, at or above the item;
  - an exclude at, above or below it;
  - a new claim above it, for each matcher kind: known paths, known children, name with siblings, contains-file, `CACHEDIR.TAG`, `.app` suffix;
  - a lock file appearing or disappearing;
  - a sibling version added or removed (toolchain, JetBrains);
  - the item swapped (`dev`, `ino`);
  - a project manifest appearing inside;
  - `.git` appearing above;
  - a picked user-data entry that became another rule's finding, or holds a keep marker;
  - the one case whose reason may differ (gate decision 1): a claim that appeared above the item, with a keep marker in another of its subtrees, is refused `kept` by the full re-check and `changed-since-scan` by the pruned one.

  Evidence: `crates/ddc-core/tests/recheck.rs`, one test per change, each comparing the whole plans: `a_keep_marker_in_at_or_above_the_item`, `an_exclude_at_above_or_below_the_item`, `a_claim_above_the_item_for_each_matcher_kind`, `a_content_claim_appearing_above_the_item`, `a_lock_file_appearing_or_disappearing`, `a_sibling_version_added_or_removed` (toolchain items always take the full walk), `the_item_swapped_a_manifest_inside_or_git_above`, `a_picked_entry_that_became_a_finding_or_holds_a_keep_marker`, `a_new_claim_with_a_marker_elsewhere_refuses_the_item_for_another_reason`.
- [x] **AC 5.** The equivalence tests pass for every rule in the catalog. — evidence: `crates/ddc-core/tests/recheck.rs` `every_rule_unchanged_is_planned_alike_and_runs` and every `each_rule` test iterate `RULES` through `crates/ddc-core/tests/common/fixtures.rs` (`rule_fixture`, which fails on a rule without a fixture); `make check` 2026-10-25
- [x] **AC 6.** After a clean, `last-scan.json` (tests):
  - lists no deleted item, and its totals dropped by that item's bytes, files and directories;
  - lists no trashed item, with its bytes moved to the `trash` finding;
  - shows a tool-cleaned item at its new size;
  - leaves skipped and not-run items as they were;
  - has lost a picked entry from its folder's contents, together with the findings inside it; the folder's bytes dropped by the entry's less those findings', and the scanned total is unchanged when the entry went to the `trash` finding;
  - has lost a deleted or trashed finding inside a listed Downloads entry from that entry's `bytes` and `findings`, or from the folder's rest;
  - is unchanged for an item the report does not hold, and never wraps a total below zero for an entry that grew since the scan.

  `generated_at` is the previous report's in the pruned case and the rescan's in the full-walk case; the same holds after a cancel.

  Evidence: `crates/ddc-core/tests/after_clean.rs` (10 tests, one per bullet, each checking the finding-derived totals against an independent recount); `crates/ddc-cli/tests/clean_items_cli.rs` `a_pruned_clean_edits_the_report_it_read`, `a_tool_cleaned_item_is_saved_at_its_new_size`, `sigterm_or_sighup_during_a_tool_ends_the_tool_and_ddc_with_the_report` (saved after a cancel, `generated_at` kept).
- [x] **AC 7.** The app starts no scan after a clean that wrote the report (after a crash that followed the `cleaning` event it rescans, as today), and its sheet moves from preparing to cleaning on the CLI's `cleaning` event. A crash after the event points to the journal; a refusal, or a crash before the event, says "Nothing was touched". A clean report with `last_scan_saved: false`, none after the event, or a saved report the app cannot read, rescans as today (DDCKit tests). On the CLI side, `clean --items --progress json` on a temporary tree writes `preparing` then `cleaning` to stderr, each flushed, with `cleaning` before the first journal entry; without `--progress json`, stderr carries no JSON (test). — evidence: `app/Packages/DDCKit/Tests/DDCKitTests/CleanRunTests.swift` `flow`, `events`, `crashAfterCleaningEvent`, `crashBeforeCleaningEvent`, `notSavedRescans`, `cancelRunning`, `refusedBeforeTouching`; `PopoverModelTests.reloadAfterClean`; `ContractTests.cleanContracts`; `crates/ddc-cli/tests/clean_items_cli.rs` `a_clean_emits_its_phases_walks_and_items_in_order`, `cleaning_is_on_stderr_before_the_first_action`, `without_the_flag_no_event_reaches_stderr`; `make app-test` 142 passed
- [x] **AC 8.** The terminal's `ddc clean` still rescans the whole home before its plan (test). — evidence: `crates/ddc-cli/tests/clean_cli.rs` `the_terminal_clean_still_rescans_the_whole_home`
- Manual checks (1) moved to [Owner checks: manual acceptance across stories](../../../ops/owner-checks-manual-acceptance-across-stories.md) on 2026-10-30, by the owner's decision.

## Final Summary

**What changed** (code commit f55b514, released in 0.2.4 with a95d442):
- `clean --items` re-checks only the listed items, their ancestors and, for a `Siblings` rule, the siblings. A `Tree` rule (nvm, pyenv, rustup), an unreadable ancestor, or a report of other roots or another home takes the full walk.
- The CLI edits `last-scan.json` after the clean so it describes the disk, and reports `last_scan_saved`. The app runs no scan after a clean that saved the report.
- The sheet moves from preparing to cleaning on the CLI's `cleaning` event. The app logs a `clean timings` line (`CleanTimings.swift`) for AC 9.
- The terminal's `ddc clean` still rescans the whole home.

**Tests run:** `crates/ddc-core/tests/recheck.rs` and `after_clean.rs`; `crates/ddc-cli/tests/clean_items_cli.rs` and `clean_cli.rs`; DDCKit `CleanRunTests`, `PopoverModelTests.reloadAfterClean`, `ContractTests.cleanContracts`; `make check` (2026-10-25) and `make app-test` (142 passed).

**Remaining risk:** AC 9, the timing on the reference machine (re-check under 1 s, first action under 3 s), moved to the owner list; until it runs, the speed-up is not measured on real data. The edited report drifts from the disk in a few figures until the next full scan (decision 6).

## Technical Notes

- **The full walk and its reason** (before this story): `crates/ddc-cli/src/cmd/clean_items.rs`, "The full rescan: only a whole-tree walk sees toolchain pins and new claims above an item". It writes `last-scan.json` before `plan_listed` and `execute`.
- **Cross-tree inputs today.**
  - `toolchain::dropped` (`crates/ddc-core/src/rules/toolchain.rs`): `toolchain.nvm`, `toolchain.pyenv` and `toolchain.rustup` need `Pins` (`crates/ddc-core/src/rules/visitor.rs`), read from pin files anywhere outside a claimed artifact. With one claimed version or none, every version is dropped, so the siblings matter too.
  - `ide.jetbrains-old-data` needs only its siblings.
  - Nothing else in the finding path reads outside the item, its parent and its ancestors:
    - name-with-sibling matchers stat the parent;
    - content matchers read inside the item;
    - excludes are path prefixes;
    - hard links seen outside the owner count as shared in both walks;
    - "inside another item" compares the list with itself.
  - Project activity only feeds the default selection, which `plan_listed` does not read. What matters above an item is the lock file (class) and `.git` (project root).
- **User-data entries** (`plan_listed`'s entry path, `contents::attach`). The `EntryIndex` comes from the walker listing the folder with times and measuring its children. The dry run's item walk is markers-only and cannot see an entry that became another rule's finding.
- **Pruned-walk caveats.**
  - (Superseded 2026-10-25 by gate decision 1: the walk stops there.) A claim on an ancestor makes the walker measure that ancestor's whole subtree, because the walker's hooks are off inside claims. Examples: a `.ddc-keep` in `~/Code`, or the Downloads region that lists entries. The plan gate decides whether to stop descent there or to take the full walk.
  - (Superseded 2026-10-25 by gate decision 2: they count nowhere.) Pruned directories raise the root's `skipped` counts. They must not read as `incomplete` for a listed item; only the item's own counts matter.
- **The keep-marker walk** (`keep_markers`, `crates/ddc-core/src/exec/mod.rs`) walks every chosen item inside `execute`, seconds before acting. If the pruned walk supplies the markers instead, they are taken a little earlier. This is safe because the guard checks a marker at the item and above at the moment of acting; only markers deep inside come from the walk. Otherwise both walks stay, and their cost is bounded by the items' size.
- **The app's signals today** (`CleanRun.swift`): it watches the modification time of `last-scan.json` to switch from preparing to cleaning, and reads "nothing was touched" from an unchanged file. `CleanRunTests` encodes both, including a crash after the save. `PopoverModel.refresh` does not reload the report during a clean.
- **The journal cannot be the crash signal.** An entry is appended after each outcome, so a crash during the first item's rename leaves it unchanged.
- **"CLI: progress events".** Its preparing counter is "files read against the last report's `totals.files`". After this story, the denominator is the listed items' files (plus `KnownChildren` siblings) in the pruned case, and `totals.files` only in the full-walk case. That story's event and schema are used as they are (note of 2026-10-25); its preparing denominator changes here.
- `ddc schedule status` reads the report's `generated_at` and selected reclaimable bytes; both stay meaningful in an edited report.

## Dependencies

- The ADR amendments above.
- "CLI: clean an explicit item list" (DDC-002): the `--items` contract this story changes.
- "CLI: progress events" (DDC-002): its `cleaning` event and `docs/progress.schema.json`, used as they are (note of 2026-10-25); its preparing denominator changes here.
- "Review and clean window" and "Menu bar popover" (DDC-002): their text about the rescan after a clean.

## Attachments

-

---

*Last updated: 2026-10-30*

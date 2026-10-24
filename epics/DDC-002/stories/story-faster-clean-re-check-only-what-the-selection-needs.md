---
type: story
id: "story-faster-clean-re-check-only-what-the-selection-needs"
epic: "DDC-002"
title: "Faster clean: re-check only what the selection needs"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-21
updated: 2026-10-24
external_refs: {}
tags: [cli, app, safety, performance]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Faster clean: re-check only what the selection needs

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
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

   It uses **one classification path**: the same rule visitor, classification and `plan_listed` as today, run over a walk pruned to those directories. A picked entry of a user-data folder also gets its folder listed with times, so its `EntryIndex` exists as it does after a full walk. Two cases fall back to the full walk without a safety reason: an unreadable ancestor, and no readable report. A rule the catalog lacks is refused as `rule-mismatch` without a walk; the scope field is exhaustive, so no rule's scope is unknown.

   The per-item checks before each action stay as they are.
2. **No second scan after the clean.** The CLI leaves `last-scan.json` describing the disk after the clean (section below), and the app runs no scan of its own after a clean that wrote it. After a crash that followed the `cleaning` event, it still rescans, as today.

These options were weighed and set aside for now:
- **Re-checking through the FSEvents history**, rescanning only the directories changed since the scan: the fastest, but it needs the walk's state persisted between runs. It is the most complex option.
- **Deleting in parallel:** in the measurement above deletion was not the bottleneck. It is worth measuring on a very large `node_modules` first.

### `last-scan.json` after a clean

**Which file.** The CLI edits the report on disk, under its exclusive lock, whichever scan wrote it: the one the user reviewed, or a newer one from a scheduled or terminal scan in between. With no report of the current schema it takes the full walk and writes a fresh report, as today.

**Per outcome:**

| Outcome | What happens in the report |
|---|---|
| Deleted | The item leaves, with its bytes, files and directories in every total it counted in. |
| Trashed | The item leaves. When the report holds the `trash` finding (`~/.Trash`) and the item lies on the home volume, its bytes, files and directories move to that finding: the scanned total does not drop and the `review` class grows. Otherwise the item left the scan roots (the Trash of another volume, or a home outside the roots) and is treated as deleted. |
| Run by its tool | The item is measured again and the difference subtracted. |
| Skipped, failed, not run | The item stays as it was. |
| Picked user-data entry | The entry leaves its folder's `contents` (entries, or the rest count and bytes). The folder finding shrinks by its bytes, and the findings inside the entry leave with it. Its bytes follow the Trashed rule. |

Every total of the report has a rule: scanned bytes, files, directories, unowned bytes, per-class count, bytes and selected figures, and the volume's free space (taken from the clean's own `statfs`).

**What stays.** In the pruned case `generated_at` stays the previous report's, so the popover's "last scan" age and `ddc schedule status` do not claim a fresh scan of the whole home. The app's "Cleaned just now" lock then lasts until the next real scan; the cleaned findings are gone from the file anyway. `duration_ms` and `ddc_version` stay the previous report's too. In the full-walk case all three are the rescan's, as for any full scan.

**When it is written.** Once, after the clean, in both the scoped and the full-walk case: today's full walk writes it before the first action, and that moves too. After a cooperative cancel it is written with what was done, and the not-run items stay. A crash or SIGKILL leaves the file from before the clean, and the app rescans as today when it saw the `cleaning` event.

**The signal that acting begins.** The app can no longer read anything from the file's date. The CLI writes the `cleaning` phase event of story "CLI: progress events" to stderr, flushed, under `--progress json`. It writes it after the journal is open and the signal handler installed, immediately before `exec::execute`, so an exit before that point (such as "cannot open the action journal") is "Nothing was touched". Only the phase events (`preparing`, `cleaning`) are brought forward here, without counters, in a first `docs/progress.schema.json` that "CLI: progress events" extends. `--progress json` is accepted by `clean` as a whole, the terminal's included, with phase events only.

The app reads both of its signals from that event:
- the switch from preparing to cleaning;
- after a crash: event seen → "the journal lists what was done"; not seen → "Nothing was touched".

A crash between the event and the first action errs on the safe side: it points to the journal although nothing was done.

The app takes the event from the runner's live `stderrLine` callback, never from `Result.stderrTail`, a bounded tail that warnings or "stopping after the current item" can push it out of. The signal does not survive an app relaunch during a clean. That is acceptable: quitting cancels the clean and waits for the CLI (`prepareToQuit` in `CleanRun.swift`), as today, and the app never resumes watching a clean it did not start.

### Scope

- The terminal's `ddc clean`, with or without `--yes`, keeps the full walk and its order.
- Story "Automatic cleaning of download caches after a scheduled scan" uses `clean --yes` with `--group` and assumes a rescan after it. It is re-read when this story lands; it is not changed here.
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

- [ ] Time each step before the first action in the app log or behind a debug flag: rescan, keep-marker walk, local snapshots, process snapshot. This confirms the rescan is the 29 s before changing it. If another step is the cost, it becomes its own story and AC 9 is re-cut.
- [ ] Amend the ADRs and the texts listed above; critic.
- [ ] Core: give every rule a scope, matched exhaustively (`Item`, `Siblings` or `Tree`), so a new cross-tree rule cannot be missed silently; a list holding a `Tree` rule takes the full walk.
- [ ] Core: the pruned walk, a visitor wrapper that skips any directory that is not an ancestor of, inside, or a sibling-for-`KnownChildren` of a listed item. It reuses the rule visitor, classification and `plan_listed` unchanged, and builds the `EntryIndex` of a picked entry's folder.
- [ ] Equivalence tests over synthetic homes covering every rule in the catalog: the pruned and the full re-check give the same outcome for every listed item, for each change of AC 4.
- [ ] CLI: the `preparing` and `cleaning` phase events under `--progress json`, with a first `docs/progress.schema.json` holding the phase events only; `last-scan.json` edited after the clean, per the section above.
- [ ] App:
  - drop the rescan after a clean that wrote the report; keep it after a crash that followed the `cleaning` event;
  - read the phase and "nothing touched" from the event;
  - reload the report when the clean ends (`PopoverModel.refresh` skips it during a clean today);
  - rewrite the crash tests.
- [ ] README and release notes.

## Implementation Plan

## Acceptance Criteria

- [ ] **AC 1.** With no `Tree` rule in the list, `clean --items` opens no directory outside the listed items, their ancestors, and the siblings of a `KnownChildren` item with their subtrees. A test asserts the opened set on a synthetic home with a large unrelated tree. An unreadable ancestor falls back to the full walk (test).
- [ ] **AC 2.** With a `toolchain.nvm`, `toolchain.pyenv` or `toolchain.rustup` item in the list, it walks the whole tree, and a pin file added after the scan protects that version (test).
- [ ] **AC 3.** A list with no report of the current schema on disk takes the full walk and writes a fresh report (test).
- [ ] **AC 4.** After the scan, each of these changes gives the listed item the same outcome as the full rescan gives today (tests):
  - a keep marker in, at or above the item;
  - an exclude at, above or below it;
  - a new claim above it, for each matcher kind: known paths, known children, name with siblings, contains-file, `CACHEDIR.TAG`, `.app` suffix;
  - a lock file appearing or disappearing;
  - a sibling version added or removed (toolchain, JetBrains);
  - the item swapped (`dev`, `ino`);
  - a project manifest appearing inside;
  - `.git` appearing above;
  - a picked user-data entry that became another rule's finding, or holds a keep marker.
- [ ] **AC 5.** The equivalence tests pass for every rule in the catalog.
- [ ] **AC 6.** After a clean, `last-scan.json` (tests):
  - lists no deleted item, and its totals dropped by that item's bytes, files and directories;
  - lists no trashed item, with its bytes moved to the `trash` finding;
  - shows a tool-cleaned item at its new size;
  - leaves skipped and not-run items as they were;
  - has lost a picked entry from its folder's contents, together with the findings inside it.

  `generated_at` is the previous report's in the pruned case and the rescan's in the full-walk case; the same holds after a cancel.
- [ ] **AC 7.** The app starts no scan after a clean that wrote the report (after a crash that followed the `cleaning` event it rescans, as today), and its sheet moves from preparing to cleaning on the CLI's `cleaning` event. A crash after the event points to the journal; a refusal, or a crash before the event, says "Nothing was touched" (DDCKit tests). On the CLI side, `clean --items --progress json` on a temporary tree writes `preparing` then `cleaning` to stderr, each flushed, with `cleaning` before the first journal entry; without `--progress json`, stderr carries no JSON (test).
- [ ] **AC 8.** The terminal's `ddc clean` still rescans the whole home before its plan (test).
- [ ] **AC 9.** On the reference machine, with the app log's times (manual check by the owner):
  - a clean of caches only re-checks its items in under 1 s;
  - the whole time before the first action stays under 3 s.

## Final Summary

## Technical Notes

- **The full walk and its reason:** `crates/ddc-cli/src/cmd/clean_items.rs`, "The full rescan: only a whole-tree walk sees toolchain pins and new claims above an item". It writes `last-scan.json` before `plan_listed` and `execute`.
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
  - A claim on an ancestor makes the walker measure that ancestor's whole subtree, because the walker's hooks are off inside claims. Examples: a `.ddc-keep` in `~/Code`, or the Downloads region that lists entries. The plan gate decides whether to stop descent there or to take the full walk.
  - Pruned directories raise the root's `skipped` counts. They must not read as `incomplete` for a listed item; only the item's own counts matter.
- **The keep-marker walk** (`keep_markers`, `crates/ddc-core/src/exec/mod.rs`) walks every chosen item inside `execute`, seconds before acting. If the pruned walk supplies the markers instead, they are taken a little earlier. This is safe because the guard checks a marker at the item and above at the moment of acting; only markers deep inside come from the walk. Otherwise both walks stay, and their cost is bounded by the items' size.
- **The app's signals today** (`CleanRun.swift`): it watches the modification time of `last-scan.json` to switch from preparing to cleaning, and reads "nothing was touched" from an unchanged file. `CleanRunTests` encodes both, including a crash after the save. `PopoverModel.refresh` does not reload the report during a clean.
- **The journal cannot be the crash signal.** An entry is appended after each outcome, so a crash during the first item's rename leaves it unchanged.
- **"CLI: progress events".** Its preparing counter is "files read against the last report's `totals.files`". After this story, the denominator is the listed items' files (plus `KnownChildren` siblings) in the pruned case, and `totals.files` only in the full-walk case. That story also depends on DDC-001 "Progress output while ddc clean runs", still planned, which is why the phase events are brought forward here.
- `ddc schedule status` reads the report's `generated_at` and selected reclaimable bytes; both stay meaningful in an edited report.

## Dependencies

- The ADR amendments above.
- "CLI: clean an explicit item list" (DDC-002): the `--items` contract this story changes.
- "CLI: progress events" (DDC-002): its phase events, brought forward here without counters; its preparing denominator changes.
- "Review and clean window" and "Menu bar popover" (DDC-002): their text about the rescan after a clean.

## Attachments

-

---

*Last updated: 2026-10-24*

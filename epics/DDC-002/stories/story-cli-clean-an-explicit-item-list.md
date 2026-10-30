---
type: story
id: "story-cli-clean-an-explicit-item-list"
epic: "DDC-002"
title: "CLI: clean an explicit item list"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-18
external_refs: {design_canvas: "https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV"}
tags: [cli, contract, json, cleanup]
code_refs: ["crates/ddc-cli/Cargo.toml", "crates/ddc-cli/src/cli.rs", "crates/ddc-cli/src/main.rs", "crates/ddc-cli/src/exit.rs", "crates/ddc-cli/src/prompt.rs", "crates/ddc-cli/src/cmd/clean.rs", "crates/ddc-cli/src/cmd/clean_items.rs", "crates/ddc-cli/src/cmd/mod.rs", "crates/ddc-cli/tests/clean_cli.rs", "crates/ddc-cli/tests/clean_items_cli.rs", "crates/ddc-cli/tests/scan_cli.rs", "crates/ddc-core/src/plan.rs", "crates/ddc-core/src/exec/", "crates/ddc-core/src/rules/classify.rs", "crates/ddc-core/src/rules/mod.rs", "crates/ddc-core/src/scan/walker.rs", "crates/ddc-core/src/scan/mod.rs", "crates/ddc-core/src/platform/macos/mod.rs", "crates/ddc-core/src/platform/macos/signals.rs", "crates/ddc-core/tests/clean_items.rs", "crates/ddc-core/tests/clean_safety.rs", "crates/ddc-core/tests/common/", "docs/clean.schema.json"]
specs: []
started_at: "2026-10-18T12:19:22.355Z"
closed_at: "2026-10-18T13:36:15.942Z"
plan_updated_at: "2026-10-18T12:19:22.355Z"
---

# CLI: clean an explicit item list

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | done |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The app cleans exactly the items the user confirmed in its sheet. This story gives the CLI that entry point: an explicit item list, a JSON report, a read-only dry run for the sheet's warnings, cancellation, and exit codes the app can act on. It is §6 items 1, 2, 4 and 5 of the ADR "Menu bar app: Swift shell over the ddc CLI", built and tested in Rust without the app.

One change is visible in the terminal: a held lock exits 75 instead of 1. The existing refusals keep exit code 2.

## Decomposition

Rough until the plan gate.

- [x] `clean --items <file|->`:
  - JSON input `{schema_version, items: [{path, dev, ino, rule, class, action, permanent?}]}`, `action` one of `delete`, `trash`, `native`, `native-only`, and `permanent` only on `review` items;
  - it rescans as every clean does, and runs a listed item only if the fresh plan finds it again with the same path, `(dev, ino)`, rule and class, and derives the same action for the item's `permanent`; anything else is skipped as `changed-since-scan`;
  - the list is the selection: no class filter, no default selection, no prompt (ADR §7); `disabled_rules` never drops a listed item;
  - refused with exit 2: a global `--permanent` together with `--items`, and a malformed list
- [x] `clean --json`: the execution report on stdout, with `schema_version`: the outcome per item, skip reasons, free space before and after for each volume touched, the journal path. Its schema is generated into `docs/clean.schema.json`
- [x] `clean --items … --dry-run --json`, for the sheet's warnings, without a rescan (ADR §6 item 2):
  - the executor's per-item re-checks against the listed items only: identity (`(dev, ino)` at the path), class and action derived again from the rule and the project's lock file, kept (`.ddc-keep`, an exclude), in use, owner running, protected;
  - no walk of any unlisted tree;
  - one JSON line per item on stdout as its checks finish, then a summary line;
  - would delete, would trash, would run a command, or would skip with its reason (`owner-running`, `in-use`, `changed-since-scan`, …);
  - no rename, no delete, no Trash, no native command
- [x] SIGINT and SIGTERM:
  - a scan needs no handler: `last-scan.json` is only ever replaced atomically at the end;
  - a clean installs a handler for both (in the core's `sys` module, the only one allowed `unsafe`, or through `signal-hook`) that sets a cooperative flag `exec::execute` checks between items;
  - a native command runs in its own process group (`CommandExt::process_group`, stable std, no `unsafe`); the handler forwards SIGINT and SIGTERM to that group (`-pgid`), so Ctrl-C in a terminal still reaches it;
  - the clean stops after the current item, and the journal holds every completed action
- [x] Exit codes: 0 done, 1 error, 2 refused (the existing refusals and the new `--items` ones), 75 another scan or clean holds the lock. With `--json`, a run that produced a report exits 0 even when items failed; their outcomes are in the report
- [x] Plan gate: decide between the full rescan (about 40 s) and a scoped re-walk of the listed findings (ADR, Open Questions); the full rescan stays unless the scoped one is shown to give the same safety

## Implementation Plan

No spec covers this story; the ADRs "Menu bar app: Swift shell over the ddc CLI" (§6, §7) and "Cleanup safety model: risk classes, plan and confirmation" (with its amendment) are the contract. DDC-001's layout is followed: the safety logic stays in `ddc-core/src/exec/` behind ports with fakes, and a mode of a command gets its own file in `ddc-cli/src/cmd/` (as `scan --background` has `background.rs`).

**Plan-gate decision: the full rescan stays** (epic and ADR Open Questions). The executor's guard already re-checks each item: roots, identity of the item and its parent, `Catalog::verify`, `.git` and project roots, `.ddc-keep`, owner running, in use. `--exclude` needs no walk either: it is a path-prefix test. What only a whole-tree walk establishes:
- the toolchain pins. `toolchain::dropped` needs every `.nvmrc`, `.python-version` and `rust-toolchain` the walk saw, and every sibling version; the guard never re-evaluates it, so a version pinned in any project after the review is protected only by a full walk;
- the ancestor claims: a rule that would now claim a directory above the item (a new cache or environment around it), which absorbs the item into a larger finding and changes its rule.

A scoped re-walk would therefore need a second classification path and would still miss pins outside the walked subtrees. The scoped per-item checks serve the dry run only, which predicts and never replaces the clean's rescan.

**Core (`crates/ddc-core`).**
- `plan.rs`:
  - `action_for(rule, class, permanent) -> Option<Action>`, extracted from `build_plan`, is the one derivation used by the plan, the item matching and the dry run;
  - the input types (`ListedItems { schema_version, items }`, `ListedItem { path, dev, ino, rule, class, action, permanent }`, `ListedAction` = `delete | trash | native | native-only`);
  - `plan_listed(&ScanReport, &Catalog, &[ListedItem])`. It matches against `report.findings` directly, not through `build_plan`, which drops user-data. For each item in input order:
    1. no fresh finding with the same path, `(dev, ino)`, rule and class: skipped as `changed-since-scan`;
    2. `action_for` gives no action (user-data, `none`, an empty native-only): skipped as `action-not-allowed`;
    3. the derived action differs from the listed one: skipped as `changed-since-scan`;
    4. otherwise a selected plan item.

    `selected` and `disabled_rules` are never read, so neither can drop a listed item.

  The listed `action` is the effective action, and `action_for` is exhaustive over removal kinds, not only over today's catalog:

  | Rule removal | Fresh class | `permanent` | Action |
  |---|---|---|---|
  | delete | safe, rebuildable | false | `delete` |
  | delete or trash | review | false / true | `trash` / `delete` |
  | trash | safe, rebuildable | false | `trash` |
  | native | any | false | `native` (fallback decided at run time) |
  | native-only (non-empty) | any | false | `native-only` |
  | none, empty native-only, user-data | – | – | `action-not-allowed` |

  `permanent: true` is valid only on an item listed with class `review` and action `delete`, and a `review` item listed as `delete` must carry it; the input check enforces both without the catalog.
- `exec/mod.rs`:
  - `execute` takes `Consent::Listed` (ADR §7), accepted like `Interactive`;
  - it takes a cancel flag (`&AtomicBool`), checked before each item, between the commands of a native-only item, after a native command's location probe, and before any fallback. A probe or command killed by the forwarded signal fails, and without these checks that failure would read as "location mismatch" and fall back to a delete or a Trash; with them the item is not run;
  - an item left by a cancel gets `Outcome::NotRun`, is never journaled, and the report says `cancelled`. An item whose command was interrupted is journaled with that command's result (`success: false`);
  - free space is measured per volume touched (`statfs` on each item's volume), not only the home's;
  - hardening: an empty native-only command list is refused, never reported as "ran tools".
- `exec/report.rs`: `CleanReport` with its own `CLEAN_SCHEMA_VERSION = 1`:

  ```
  { schema_version, ddc_version, cancelled,
    items: [{ path, dev, ino, rule, class, action, permanent, bytes?, <outcome> }],
    volumes: [{ mount_point, free_before, free_after }],
    local_snapshots, journal }
  ```

  `Outcome`, `SkipReason` and `CommandResult` get `JsonSchema` and `Deserialize`. The scan's `SCHEMA_VERSION` does not change, and `docs/scan.schema.json` stays byte-identical.
- `exec/preflight.rs`: the dry run. It has one `Guard` from one process snapshot. For each item in turn:
  1. the exclude check (a path-prefix test, as in the scan);
  2. the class and action derived again (`effective_class`, extracted from `classify.rs`, then `action_for`);
  3. a walk of the item alone through an injected `DirReader`, for markers, `incomplete` and bytes;
  4. `Guard::check_with_markers`;
  5. for a native item, the tool looked up (`find`), never run.

  After each item it writes and flushes `{"type":"item", …, "prediction": "would-delete|would-trash|would-run|would-skip", "reason"?}`, and it ends with `{"type":"summary", …}`. Its ports have no Trasher and no Journal, so it cannot act by construction. It takes the shared lock and skips `tmutil`.
- `scan/walker.rs`: `walk_with_reader(.., &dyn DirReader)`, so a test can count the directories opened.
- `sys/signals.rs`, in the `sys` module, the one place with `unsafe` (`libc` is already a dependency):
  - a `sigaction` handler for SIGINT and SIGTERM, installed with `SA_RESTART`, stores the flag and the signal number in atomics, and forwards the signal with `kill(-pgid, sig)` only when a native command's group is published (a zero group is never signalled: `kill(0, sig)` would hit ddc's own group, under `cargo test` the harness). All of these are async-signal-safe;
  - `set_child_group`, `clear_child_group`, `kill_group`;
  - `reraise()`: restores `SIG_DFL` for the signal, unblocks it and raises it, so the process ends by that signal.
- `exec/system.rs::run`:
  - spawns with `process_group(0)` and publishes the child's group right after `spawn`, then re-checks the flag, so a signal arriving in between is not lost;
  - on timeout it kills the whole group, which also fixes a hang where a grandchild keeps the pipes open;
  - it clears the group once the child is reaped.

**CLI (`crates/ddc-cli`).**
- `cli.rs`:
  - `--items <FILE|->`, conflicting with `--permanent`, `--yes`, the class and category filters and `--min-size`; clap's usage errors exit 2 before anything runs;
- `cmd/clean.rs` dispatches `--items` to a new `cmd/clean_items.rs`, after its root check (so root still exits 2):
  - a malformed list is refused (exit 2) before the lock, the environment and the journal are opened: wrong `schema_version`, an unknown field, an empty list, a duplicate path, a path that is not absolute and normalised (compared byte for byte, never canonicalised), `permanent` on a non-review item;
  - otherwise it locks, rescans with the roots and excludes the app passes as positional paths and `--exclude` (as for `clean`), saves the rescan before executing (as `clean` does today, so a cancelled run has saved it), plans with `plan_listed`, installs the signal handler, executes with `Consent::Listed`, and prints the `CleanReport` on stdout;
  - exit status with `--json`: 0 whenever the run produced a report, failed items included (the report holds each item's outcome and a `failed` count); 1 only when no report could be produced. The app decodes stdout on 0 and on `.cancelled`;
  - nothing else goes to stdout with `--json` (the plan, the Time Machine note and the summary go to stderr or are skipped);
  - no terminal is required.
- Plain `ddc clean` installs the same handler right before `execute`, so Ctrl-C stops it after the current item.
- When a signal arrived at any point after the handler was installed, ddc writes its report, flushes stdout (the journal is synced per line already), and then calls `reraise()`, so the app sees `.cancelled` (DDCKit maps only an uncaught signal to it) and still decodes stdout. A signal during the rescan, before the handler, ends ddc by the default action with empty stdout; the app story must not expect a report then.
- `main.rs`: `LockError::Busy` exits 75 (`BUSY`), refusals exit 2 (`REFUSED`); `scan --background` keeps exiting 0 on a busy lock.
- `--json` requires `--items`. `--dry-run` with `--items` is the preflight; `--dry-run` without it keeps today's text plan, and its test keeps passing.
- `docs/clean.schema.json` is generated, with the `DDC_UPDATE_SCHEMA` test pattern of `docs/scan.schema.json`. The input list and the dry-run lines get their schemas in the same file's definitions, because the app decodes strictly.

**Tests**, per criterion:
- The new core tests go in `crates/ddc-core/tests/clean_items.rs`; the fakes of `clean_safety.rs` move to `tests/common/`.
- AC1, end to end in `clean_cli.rs`: `scan --json` gives `dev`/`ino`; one listed directory is replaced after the scan; `~/Pictures` is also listed, as `delete`.
- AC2, core only: the real `SystemTrash` ignores `HOME`, so a fake Trasher is used.
- AC3 and AC4, end to end: the tree's snapshot is unchanged, the data directory is absent, and stdin and stdout are `Stdio::null()`.
- AC5 and AC6, core: `FakeProcs` reports an owner running, `FakeCommands` records no call, and a counting `DirReader` wraps the real one. The dry run's preflight is given the same roots as the clean. AC6 asserts that every directory the reader opened lies inside a listed item; the single-directory probes outside (the parent and project root for lock files, ancestors for `.ddc-keep` and `.git`) do not go through the reader and are named in the test.
- AC7: the schema test, then a round trip of the output through `CleanReport` (`jsonschema` is not available offline), with one volume.
- AC8, core: a journal fake sets the flag on its first record. A second core test: `FakeCommands` sets the flag during a native item's location probe; the item ends as not run, and nothing is deleted or trashed.
- AC9, end to end: a fake `uv` on `PATH` (`uv cache dir` prints the temp cache; `uv cache clean` writes its pid and sleeps 30 s); the test's `ddc()` helper sets `PATH` to the fake's directory, because `SystemCommands::find` searches `Env::tool_dirs()`. `kill -TERM` goes to ddc after the pid file appears; ddc ends within a second by signal 15, and the tool's pid is gone.
- AC10, end to end: `/usr/bin/script -q /dev/null ddc clean --yes …` gives a pseudo-terminal; `--yes` reaches the fake tool through `cache.uv` (safe, selected by default, `~/.cache/uv`). BSD `script` forwards its stdin to the pseudo-terminal and sends end-of-file when stdin closes, so stdin stays an open pipe, and `\x03` is written only after the pid file appears. `script` exits with the child's signal number (2 for SIGINT, the same as "refused"), so the test asserts on the tree, the journal and the tool's pid, never on `script`'s status. If `script` proves unreliable, the fallback is a `process_group(0)` spawn and `kill -INT -<pgid>`.
- AC11, end to end: a pre-seeded `last-scan.json`; ddc is spawned, stopped at once with SIGSTOP, sent SIGTERM, then SIGCONT, so the walk cannot finish first; `last-scan.json` is byte-identical afterwards and no `.last-scan.json.tmp-*` remains.
- AC12: the existing lock test asserts 75; a new scan test holds an exclusive lock (a shared one does not conflict).

**Order of work.**
1. Exit 75.
2. `action_for` and `effective_class`.
3. The input contract and `plan_listed`.
4. The cancel flag and `NotRun`.
5. Signals and process groups.
6. The report and its schema.
7. `clean_items.rs`.
8. The preflight.
9. The tests.
10. `make check`.

**Pitfalls.**
- The journal records an interrupted item with its failed command result and never records a not-run item; AC9 and AC10 are read that way.
- `keep_markers` checks the cancel flag between targets.
- The story "Progress output while ddc clean runs" also changes `execute`'s signature, so `execute` takes an options struct both can extend.
- The app must pass the reviewed scan's roots and excludes, or every item is skipped as `changed-since-scan`. (Superseded by "CLI: shared settings, types and the notify decision": the app passes none, and every command resolves them from `settings.json`.)
- `clean` saves its pre-clean rescan, so the popover lists removed items until the next scan: the app rescans after a clean.
- The dry run cannot see pins, so the real clean may skip an item the sheet predicted; the result screen lists it.
- The dry run never runs a location probe, so "would run" may execute as the rule's fallback (a delete for uv and go, the Trash for pyenv and rustup) when the tool reports another location.
- Ctrl-Z now stops ddc but not a native command in its own group; acceptable for a clean that is either running or cancelled.
- schemars must accept `Outcome::Skipped(SkipReason)` (a newtype variant whose inner enum is itself internally tagged); this is checked first when the report's schema is built.

**Proposed `code_refs`.**
- Story: `crates/ddc-core/src/plan.rs`, `crates/ddc-core/src/exec/`, `crates/ddc-core/src/sys/`, `crates/ddc-core/src/scan/walker.rs`, `crates/ddc-core/src/rules/classify.rs`, `crates/ddc-core/tests/`, `crates/ddc-cli/src/cli.rs`, `crates/ddc-cli/src/main.rs`, `crates/ddc-cli/src/cmd/`, `crates/ddc-cli/tests/`, `docs/clean.schema.json`, `README.md`.
- Epic DDC-002, new files only: `crates/ddc-cli/src/cmd/clean_items.rs`, `crates/ddc-core/src/exec/preflight.rs`, `crates/ddc-core/src/exec/report.rs`, `crates/ddc-core/src/sys/signals.rs`, `docs/clean.schema.json`.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [x] Test: `--items` with one valid item and one whose directory was swapped after the scan deletes the first and skips the second as `changed-since-scan`; a listed item of a `user-data` rule is refused (`action-not-allowed`) — evidence: crates/ddc-cli/tests/clean_items_cli.rs::listed_items_are_cleaned_and_a_swapped_or_user_data_one_is_refused; crates/ddc-core/tests/clean_items.rs::a_user_data_item_is_refused_and_a_swapped_one_is_changed_since_scan
- [x] Test: a listed `review` item with `permanent: false` goes to the Trash, and with `permanent: true` is deleted; an item listed as `review` with action `trash` that the fresh scan classifies `rebuildable` (a lock file appeared) is skipped as `changed-since-scan`, not deleted — evidence: crates/ddc-core/tests/clean_items.rs::a_review_item_goes_to_the_trash_unless_permanent_and_a_reclassified_one_is_skipped
- [x] Test: `--items` together with `--permanent` exits 2, and so does a malformed list; neither touches the tree or the journal — evidence: crates/ddc-cli/tests/clean_items_cli.rs::items_with_permanent_or_a_malformed_list_are_refused_untouched (also `--yes` and a missing file); crates/ddc-core/tests/clean_items.rs::a_malformed_list_is_refused_before_anything_runs
- [x] Test: `clean --items` runs with stdin and stdout not a terminal (no `no terminal` refusal), as ADR §7 states — evidence: crates/ddc-cli/tests/clean_items_cli.rs::listed_items_are_cleaned_and_a_swapped_or_user_data_one_is_refused (stdin null, stdout a pipe, exit 0)
- [x] Test: `--dry-run --json` over a temp tree reports would delete / would trash for its items and an `owner-running` skip for an item whose owner a fake port reports running; afterwards the tree and the journal are unchanged and the fake native runner recorded no call — evidence: crates/ddc-core/tests/clean_items.rs::the_dry_run_predicts_and_changes_nothing (with a native item: `would-run`, no call recorded, not even the probe); crates/ddc-cli/tests/clean_items_cli.rs::the_dry_run_prints_a_line_per_item_and_changes_nothing (list on stdin)
- [x] Test: with a fake walker port that counts the directories it opens, a dry run over N listed items in a tree with other, unlisted trees walks no directory outside the listed items (the only reads outside are single-directory probes of the parent and project root for lock files and of ancestors for `.ddc-keep` and `.git`, which the test names); the output is N item lines, each written before the summary line — evidence: crates/ddc-core/tests/clean_items.rs::the_dry_run_walks_only_the_listed_items_and_streams_a_line_per_item
- [x] Test: `clean --items --json` output validates against `docs/clean.schema.json`, the committed schema equals the generated one (as for `docs/scan.schema.json`), and it holds free space before and after for each volume touched; a run with a failed item exits 0 and reports the failure — evidence: crates/ddc-core/tests/clean_items.rs::committed_clean_schema_is_current (committed = generated), ::the_clean_report_round_trips_and_names_each_volume; crates/ddc-cli/tests/clean_items_cli.rs::the_report_round_trips_names_the_volume_and_a_failed_item_still_exits_0 (stdout parsed as CleanReport and serialised again equal, one volume, failed: 1, exit 0). No validator runs in the tests (none is available offline); the reviewer checked real reports against the schema with a subset validator outside the repository
- [x] Unit test: with the cooperative flag set after the first of three items, `exec::execute` returns after that item and reports the other two as not run; with the flag set during a native item's location probe, the item is not run and nothing is deleted or trashed — evidence: crates/ddc-core/tests/clean_items.rs::a_cancel_after_the_first_of_three_items_leaves_the_others_not_run, ::a_cancel_during_the_location_probe_never_becomes_the_fallback
- [x] Test: with a fake native tool on `PATH` that sleeps, SIGTERM to `ddc clean --items` ends the tool and ddc within a second, ddc ends by signal 15, and its stdout holds the report; the journal holds every started item with its result (the interrupted command with `success: false`) and no not-run item — evidence: crates/ddc-cli/tests/clean_items_cli.rs::sigterm_or_sighup_during_a_tool_ends_the_tool_and_ddc_with_the_report (fails with forwarding removed; SIGHUP too)
- [x] Test: Ctrl-C in a terminal (a pseudo-terminal, SIGINT to its foreground process group) while a fake native tool runs during `ddc clean` ends the tool, and the clean stops after that item; the journal holds the items before it and the interrupted one with its result, and no later item — evidence: crates/ddc-cli/tests/clean_items_cli.rs::ctrl_c_in_a_terminal_reaches_the_tool_and_stops_the_clean_after_it (`/usr/bin/script`; fails with forwarding removed)
- [x] Test: SIGTERM to a scan before it finishes (stopped with SIGSTOP, sent SIGTERM, continued) leaves `last-scan.json` byte-identical and no temporary file — evidence: crates/ddc-cli/tests/scan_cli.rs::sigterm_before_the_scan_finishes_leaves_the_saved_scan_untouched (covers a signal before the write begins; a signal during the write can leave the temporary file, see Risks)
- [x] Test: with the lock held, `scan` and `clean` exit 75 (the lock test in `crates/ddc-cli/tests/clean_cli.rs` asserts the code); the existing refusal tests still assert 2 — evidence: crates/ddc-cli/tests/clean_cli.rs::clean_does_not_start_while_the_lock_is_held, ::scan_does_not_start_while_a_clean_holds_the_lock (75); ::yes_with_review_class_is_rejected, ::without_a_terminal_a_flag_is_required (2)
- [x] `make check` passes — evidence: `PATH=$HOME/.cargo/bin:$PATH make check`: 138 tests, fmt and clippy clean (2026-10-18)

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

**What changed.**
- `ddc clean --items <file|->` (`crates/ddc-cli/src/cmd/clean_items.rs`) validates the list first: strict fields, normalised absolute paths, no duplicates, and `permanent` only on `review` items listed as `delete` (`ListedItems::parse`). Nothing is locked, created or journaled before that. Then it takes the exclusive lock, rescans and saves the rescan.
- `plan_listed` (`crates/ddc-core/src/plan.rs`) keeps an item only when the fresh scan has the same path, `(dev, ino)`, rule and class, and `action_for` derives the same action. Any other item is skipped as `changed-since-scan` or `action-not-allowed`, as `rule-mismatch` for an unknown rule, or as `kept` at or under a fresh `user.kept`. It runs with `Consent::Listed`.
- `--json` prints a `CleanReport` (`exec/report.rs`, schema `docs/clean.schema.json`). It holds every listed item in list order with its outcome, a `failed` count, free space before and after per volume, local snapshots, the journal path and `cancelled`.
- `--dry-run` is a read-only preflight (`exec/preflight.rs`). For each listed item it checks identity, class, action, excludes, `.ddc-keep`, the project root, a running owner and use, then walks that item alone through a `DirReader` port. It streams one JSON line per item and then a summary, and only looks tools up, never runs them.
- SIGINT, SIGTERM and SIGHUP (`sys/signals.rs`): an `SA_RESTART` handler sets a cooperative flag, forwards the signal to the process group of the native command, and writes "ddc: stopping after the current item" to stderr at once. `exec/system.rs` spawns that command with `process_group(0)`. SIGHUP is beyond the story's text: in its own group the tool would otherwise outlive a closed terminal unsupervised.
- `exec::execute` checks the flag before each item, between native-only commands, after a location probe and before any fallback. An item not started is `not-run` and is never journaled. After the report, ddc ends by the same signal (`reraise`). Plain `ddc clean` uses the same handler, so Ctrl-C there now stops after the current item.
- Visible elsewhere:
  - `ExecReport` replaced its free-space pair with `volumes` and gained `cancelled`;
  - the text output prints "Cancelled: N items were not started." and one "Free space" line per volume (`prompt.rs`);
  - a command's timeout kills its whole process group.
- Exit codes (`crates/ddc-cli/src/exit.rs`): 2 refused, and 75 when the lock is held, for scan and clean. With `--json`, the exit code is 0 whenever a report was produced.
- Extracted for reuse: `effective_class` (`rules/classify.rs`) and `walk_with_reader` (`scan/walker.rs`). The test fakes moved to `crates/ddc-core/tests/common/fakes.rs`.

**Why.** This is the CLI side of §6 items 1, 2, 4 and 5, and of §7, of the ADR "Menu bar app: Swift shell over the ddc CLI". The app's confirmation sheet hands ddc exactly the confirmed items, shows warnings before the run, can cancel it and reads every outcome. The plan gate kept the full rescan (ADR Open Questions): only a walk of the whole tree sees toolchain pins and new claims above an item.

**Deviation from the plan.** The dry run runs `Guard::check` before its own walk of the item, instead of `check_with_markers` after it. That way nothing behind a swapped path is read. The walk then finds any `.ddc-keep` inside the item.

**Tests.**
- `make check`: 138 tests pass, fmt and clippy clean (2026-10-18).
- For this story: `crates/ddc-core/tests/clean_items.rs` (10 tests), `crates/ddc-cli/tests/clean_items_cli.rs` (7), one new test in `crates/ddc-cli/tests/scan_cli.rs`, and the exit-75 assertions in `crates/ddc-cli/tests/clean_cli.rs`.
- No test runs a JSON Schema validator (none is available offline). The schema test compares the committed schema with the generated one, and the CLI test parses the real output as `CleanReport`. The reviewer checked real reports against `docs/clean.schema.json` with a subset validator kept outside the repository.
- Review fix, signal tests. `projectstore:reviewer` found that the two signal tests did not test forwarding: the fake tool's `sleep` was not on the test's `PATH`, so the tool exited 127 by itself.
  - The fake now writes its pid using only shell builtins and then runs `exec /bin/sleep 30`.
  - Why only builtins: a shell that waits for a child that exits normally ignores a SIGINT that arrived during the wait. So with a `/bin/mv` before the `exec`, the forwarded Ctrl-C was lost.
  - With forwarding removed from `sys/signals.rs`, both tests fail; the reviewer repeated this with its own mutant. With forwarding restored, they passed 30 out of 30 parallel runs here and 60 out of 60 in the reviewer's runs.
- Critic fixes: SIGHUP handled and tested with SIGTERM; the stop notice on stderr; a text-mode test (`the_text_mode_counts_refused_items_as_skipped_and_exits_1_on_a_failed_item`).
- Other review fixes:
  - the child's process group is cleared only after its pipes are drained;
  - the dry run and the clean give the same reason for a newly kept item and for an unknown rule;
  - the dry run fails on an exclude it cannot resolve, as the rescan does;
  - in text mode, `--items` counts refused items as skipped and exits 1 only when an item failed, as plain `clean` does;
  - the dry-run tests now cover a native item and a list read from stdin.

**Risks and follow-ups.**
- A signal during the rescan, before the handler is installed, ends ddc by the default action with empty stdout. It can also leave `.last-scan.json.tmp-<pid>`, which was already true before this story. The app must not expect a report in that case.
- A signal that arrives after the last item has started ends ddc by that signal, with a complete report and `cancelled: false`. So the app reads what was left undone from the report, not from how the process ended. This is noted in "Review and clean window".
- Plain `ddc clean` now stops only after the current item, so a long delete cannot be interrupted in the middle; the notice on stderr says so at once. A second Ctrl-C that exits at once is a follow-up that no story holds yet.
- An action mismatch on the app's side (a `safe` item listed as `trash`) is reported as `changed-since-scan`, as the plan states; it is a list error rather than a change.
- `CHILD_GROUP` holds one process group because commands run one at a time. A parallel runner would need more than one.
- The scan test (AC11) relies on SIGSTOP arriving before a tiny scan finishes. Across the reviewer's runs and mine, there were 0 failures in 130 runs.
- The README is not updated. The distribution story owns it, and exit code 75 and the new Ctrl-C behaviour will be documented there.

## Technical Notes

- Consent: §7 of the ADR "Menu bar app: Swift shell over the ddc CLI" makes an explicit item list interactive consent, recorded as an amendment of §5 of the safety ADR. The executor's refusals and re-checks are unchanged.
- Before this story: refusals return `ExitCode::from(2)` (`crates/ddc-cli/src/cmd/clean.rs`), errors `ExitCode::FAILURE` (`main.rs`), and a held lock surfaces as an error (1).
- The clean report and its skip reasons are new, so `changed-since-scan` adds nothing to an existing schema.
- Today a native command is spawned in the terminal's foreground process group (`crates/ddc-core/src/exec/system.rs`) and gets Ctrl-C itself. Its own process group takes it out of that group, so without forwarding Ctrl-C would stop reaching it: the forwarding of SIGINT is what prevents that regression.
- The dry run predicts; it never replaces the clean's own rescan and re-checks.
- The DDC-001 story "Progress output while ddc clean runs" left Ctrl-C between items out of scope; the SIGTERM handler here is the same mechanism and can serve SIGINT too.

## Dependencies

- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]

## Attachments

- Design canvas: https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV

---

*Last updated: 2026-10-18*

---
type: story
id: "story-whole-disk-free-space-with-the-owning-tool"
epic: "DDC-002"
title: "Whole disk: free space with the owning tool"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-30
external_refs: {}
tags: [cli, app, safety, whole-disk]
code_refs: ["crates/ddc-core/src/free/", "crates/ddc-core/src/exec/journal.rs", "crates/ddc-core/src/exec/mod.rs", "crates/ddc-core/src/exec/native.rs", "crates/ddc-core/src/disk/mod.rs", "crates/ddc-core/src/disk/model.rs", "crates/ddc-core/src/disk/simulators.rs", "crates/ddc-core/src/report/after_clean.rs", "crates/ddc-core/src/store.rs", "crates/ddc-core/src/plan.rs", "crates/ddc-core/src/lib.rs", "crates/ddc-core/tests/disk_free.rs", "crates/ddc-core/tests/common/fakes.rs", "crates/ddc-core/tests/clean_safety.rs", "crates/ddc-cli/src/cmd/disk_free.rs", "crates/ddc-cli/src/cmd/mod.rs", "crates/ddc-cli/src/cmd/clean_items.rs", "crates/ddc-cli/src/cli.rs", "crates/ddc-cli/src/render.rs", "crates/ddc-cli/src/prompt.rs", "crates/ddc-cli/tests/disk_free_cli.rs", "crates/ddc-cli/tests/structure.rs", "docs/disk-free.schema.json", "docs/disk.schema.json", "README.md", "app/Packages/DDCKit/Sources/DDCKit/FreeReport.swift", "app/Packages/DDCKit/Sources/DDCKit/AreaActions.swift", "app/Packages/DDCKit/Sources/DDCKit/DiskFreeRun.swift", "app/Packages/DDCKit/Sources/DDCKit/CLIRunner.swift", "app/Packages/DDCKit/Sources/DDCKit/DiskModel.swift", "app/Packages/DDCKit/Sources/DDCKit/DiskReport.swift", "app/Packages/DDCKit/Sources/DDCKit/DiskSummary.swift", "app/Packages/DDCKit/Tests/DDCKitTests/DiskFreeTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ContractTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/DiskSummaryTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/FakeCLI.swift", "app/Packages/DDCKit/Tests/DDCKitTests/Fixtures/free-preview.json", "app/Packages/DDCKit/Tests/DDCKitTests/Fixtures/free-report.json", "app/DeepDriveCleaner/AreaActionSheet.swift", "app/DeepDriveCleaner/DiskView.swift", "app/DeepDriveCleaner/AppDelegate.swift"]
specs: []
started_at: "2026-10-28T15:24:31.635Z"
closed_at: "2026-10-30T06:45:43.993Z"
plan_updated_at: "2026-10-28T15:24:31.635Z"
---

# Whole disk: free space with the owning tool

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

Outside the home folder, DeepDriveCleaner never removes anything by path ("Whole disk: what lies outside the home folder", DDC-002). Two areas have an owning tool that frees space safely, and this story lets the user run that tool after a confirmation:
- **Simulator runtimes**: `simctl runtime delete <identifier>`. On the reference machine two runtimes hold 16 GB.
- **Homebrew**: `brew cleanup`. It removes old versions of installed formulae and casks, stale locks, and downloads older than 120 days.

Both run only from an explicit confirmation by a person: the sheet in the app, or a prompt at a terminal. Nothing scheduled runs them, and nothing runs them unattended. The owner rejected unattended cleaning on 2026-10-28.

The story amends both ADRs before any code. The safety ADR currently says, in §6, that `brew cleanup` "is not used in v1". The menu bar ADR, in §7, names the confirmation sheet's Clean as the only caller of a destructive CLI command.

## Decomposition

Rough until the plan gate.

- [x] **Amendment to the safety ADR, "Cleanup safety model: risk classes, plan and confirmation".**
  - **§6.** Replace the `brew cleanup` sentence: `brew cleanup` runs only as the Homebrew area's action, from a confirmation.
  - **The area action, a new kind of native action.** It is not a catalog rule, and it has no path inside the scan roots. Its guard is a location probe:
    - a runtime's `parentMountPath` must lie under `/System/Library/AssetsV2`;
    - `brew --cellar` and `brew --cache` must match the Homebrew area's paths.
  - **§5.** `ddc disk free` takes the exclusive lock, because `brew cleanup` writes inside `~/Library/Caches/Homebrew`, a scan root.
  - **§7.** Every run writes an `actions.jsonl` line with the area, the action, the exact command, the bytes before and after, and the result. An area action has no `path`, `(dev, ino)`, `rule` or `class`, so this is a new line kind with a `kind` discriminator. The journal gets its own `JOURNAL_SCHEMA_VERSION`, as `clean` and `progress` have theirs; today it borrows the scan report's `SCHEMA_VERSION`, and bumping that would invalidate every `last-scan.json`.
  - **Consent.** This is a separate channel, not `--yes`, because §5 keeps `--yes` for the `safe` class without a terminal. The app passes a confirmed action list (`--actions FILE|-`), as `clean --items` does. At a terminal, a prompt asks.
- [x] **Amendment to the menu bar ADR, "Menu bar app: Swift shell over the ddc CLI".**
  - **§6:** the new commands `ddc disk`, `ddc disk free` and its `--dry-run`, with their schemas and exit codes.
  - **§7:** one caller per destructive call, extended to `disk free`. The caller is the Whole disk sheet's confirm button, and `ContractTests` gains a test that names it.
  - **§1:** `disk free` is a `CLIRunner` queue kind beside `clean`, never in the read-only `.disk` lane of "Whole disk in the app". It takes the ddc lock, so a Scan now during a long `brew cleanup` waits in the queue instead of failing busy. The two never run at once in the app: each button is disabled while the other's run is queued or running, because both write `last-disk.json` (amendment of 2026-10-28, story "Whole disk in the app"). From the terminal they can still overlap; then the last to finish writes `last-disk.json`, as "Whole disk: what lies outside the home folder" accepts.
- [x] **CLI: `ddc disk free --actions FILE|- [--dry-run] [--json]`.**
  - **Simulator runtime:**
    - It runs `simctl runtime delete <identifier>`, where `identifier` is the runtime's asset id from `runtime list -j`, not its `runtimeIdentifier`. It never passes `--keep-asset`, which would free nothing. `simctl` is run as in "Whole disk: what lies outside the home folder": never `xcrun`, never the shim under `xcode-select -p`, only the real CoreSimulator binary after `xcodebuild -checkFirstLaunchStatus` exits 0.
    - The booted check joins devices on `runtimeIdentifier`.
    - It refuses when a device on the runtime is booted, like the in-use check. `simctl runtime delete` itself would shut such devices down first; a person's running simulator is not ours to stop.
    - The location probe uses the runtime's `parentMountPath` and `path`. Both fields were read from Xcode 27's `simctl runtime list -j` on 2026-10-28, and `--keep-asset` and `--dry-run` from its usage text.
    - The dry run is `simctl runtime delete <identifier> --dry-run`, in the order of `simctl`'s usage text.
    - Deletion is asynchronous ("Deleting"). The command waits until `simctl runtime list -j` no longer lists the runtime, for at most `NATIVE_TIMEOUT`, and otherwise reports it as still deleting.
    - The freed bytes are measured, not assumed: the runtime's AssetsV2 folder before and after. When the runtime is gone but the folder did not shrink, the result says "runtime removed, Z GB not freed", so it never reports space that was not freed.
    - The sheet says that the runtime of the installed Xcode's SDK is downloaded again, several GB, at the next build for that simulator.
  - **Homebrew:**
    - It runs `brew cleanup` with no arguments, never `-s` or `--prune=all`, and with `HOMEBREW_NO_AUTOREMOVE=1`, `HOMEBREW_NO_ENV_HINTS=1` and `NO_COLOR=1`: without the first, `brew cleanup` also uninstalls formulae installed only as dependencies that nothing needs (critic, 2026-10-28). `brew` is `/opt/homebrew/bin/brew`, the Homebrew area's own, not the first on the tool path.
    - The dry run is `brew cleanup -n` with the same environment, and the sheet lists what it would remove and the estimate.
    - It touches mainly two places, and the sheet names them: the Cellar (old versions that are neither linked nor pinned) and `~/Library/Caches/Homebrew`. It also prunes `~/Library/Logs/Homebrew`, stale locks and broken symlinks in its prefix. It does not remove old cask versions from the Caskroom.
    - Started by the app, brew does not see `HOMEBREW_*` variables of the user's shell profile, such as `HOMEBREW_NO_CLEANUP_FORMULAE`; Homebrew's own `brew.env` files still apply. The sheet says so.
    - Afterwards the `cache.homebrew` finding in `last-scan.json` is measured again, as a clean does (`after_clean`), and the Homebrew area in `last-disk.json` is marked stale.
  - **Running the tools:**
    - Both tools run through the existing `SystemCommands` port with `NATIVE_TIMEOUT`, 600 s, with stdin closed.
    - When a tool fails because it needs an administrator's password or a terminal, the result carries the exact command for the user to run in Terminal. The app shows it with a Copy button.
  - **At a terminal without `--actions`**, it lists the available actions with their dry-run estimate and asks for each one, default No. Without a terminal and without `--actions`, it refuses.
  - **Output:** a `--json` report with the schema `docs/disk-free.schema.json` at version 1. The exit codes follow `clean --items`.
- [x] **App:**
  - an action button on a runtime row and on the Homebrew row;
  - a confirmation sheet with the dry run's preview and the exact command;
  - progress, then the result in the row;
  - a refused or failed action keeps its message and the Terminal command.

## Implementation Plan

Planned 2026-10-28 with the planner, against the ADR amendments of 2026-10-28 (safety ADR Amendments entry; menu bar ADR §1, §6 item 11, §7). Shape follows "Coding agents in the Whole disk view": a new top-level core module, a `disk` subcommand, a DDCKit report and model, a view part, fake-CLI and contract tests.

**Decided at this gate, where the ADRs are silent** (conservative, each reported):
- A runtime whose `path` does not lie in a `<hash>.asset` folder under the `parentMountPath` is refused `no-asset-folder`: its freed bytes could not be measured.
- "An action named twice" is the same `(action, identifier)` pair twice, or `brew-cleanup` twice. Two different runtimes in one list stay legal.
- A measured folder at or under a `settings.json` skip is never opened: its bytes are reported as not measured and the `cache.homebrew` re-measure is skipped. The action itself is not refused.
- `disk free` refuses as root, as `clean` does.
- A failing `simctl list devices -j` refuses (`devices-unknown`); it never defaults to no devices.
- simctl's and brew's dry-run text is kept as raw lines; brew's "would free approximately" is parsed leniently into an optional estimate. The runtime's estimate is its measured asset folder.
- The view's "Scan Whole Disk" button is what the ADR calls Measure.

**Core** (`crates/ddc-core`)
1. `exec/journal.rs`: `JOURNAL_SCHEMA_VERSION = 2`, `kind` on every line (`item` | `area-action`), `AreaActionEntry` and an `AreaJournal` trait on `FileJournal` and a memory journal. `model::SCHEMA_VERSION` stays 1, pinned by a test.
2. `disk/simulators.rs`: extract `locate()` (xcode-select → `xcodebuild -checkFirstLaunchStatus` → CoreSimulator's simctl, with `DEVELOPER_DIR`) and make the runtime and device parsers `pub(crate)`. `disk/mod.rs`: `Sources.brew` = `<data root>/opt/homebrew/bin/brew`. `disk/model.rs`: `Area.stale`, optional, skipped when false; `DISK_SCHEMA_VERSION` stays 1. No action code under `disk/`: `structure.rs` forbids it there.
3. New `free/`: `model.rs` (`FREE_SCHEMA_VERSION = 1`, `ActionList` with `deny_unknown_fields` and its parse errors, `FreeReport`, `ActionResult` tagged kebab-case, `Refusal` strict enum, `Preview`); `probe.rs` (the location guard, folders by `(dev, ino)`, the AssetsV2 prefix with its trailing slash, the cache compared as a path when missing); `simctl.rs` (delete, then poll the list under one shared deadline, cancel between polls); `brew.rs` (`BREW_ENV`, `cleanup` / `cleanup -n`); `measure.rs` (allocated bytes through `scan::walk`); `mod.rs` (`run`, `dry_run`, `available`, `edit_last_disk`).
4. `store.rs`: `read_last_disk` with the disk version gate. `report/after_clean.rs`: `edit_tool_cleaned` for a re-measured `cache.homebrew`, without fabricating a clean report. `exec/native.rs`: share `tail()`.

**CLI** (`crates/ddc-cli`)
5. `cli.rs`: `disk free [--actions FILE|-] [--dry-run] [--json]` (with hidden `--system-root` and `--data-dir`); no `--yes`. `cmd/mod.rs`: dispatch, and `read_list` shared with `clean_items.rs`.
6. `cmd/disk_free.rs`: parse and refuse before anything; without `--actions`, no TTY → 2, at a TTY dry-run under the shared lock and ask each `[y/N]`; signals; exclusive lock (dry run: shared); journal; run; re-measure `cache.homebrew` when `last-scan.json` has the current schema and the home's roots; edit `last-disk.json`; print; re-raise a signal. Exit codes as `clean --items`. `render.rs`, `prompt.rs`, README section.
7. `docs/disk-free.schema.json` (with `ActionList`), `docs/disk.schema.json` regenerated for `stale`.

**App**
8. DDCKit: `CLIRunner.Kind` `.diskFree` and `.diskFreeDryRun`, queue kinds, never coalesced. New `AreaActionList`, `FreeReport` (strict), `AreaActions` (which rows offer an action and the sheet's wording), `DiskFreeRun` (`extension DiskModel`, the only code that runs `disk free`: `previewAction`, `freeConfirmedActions`, `stopFree`; the list written to a 0600 temp file). `DiskModel`: free state; Scan Whole Disk disabled while a free run is queued or running, action buttons while a `disk` run goes; after a run, reload, and after `brew-cleanup` tell the popover to reload `last-scan.json`. `DiskReport.Area.stale`; `DiskSummary`: runtime sub-rows, row actions, "before cleanup" for a stale size.
9. App target: `AreaActionSheet.swift` (preview, exact command, what cannot be undone, confirm); `DiskView`: "Delete Runtime…" and "Clean Up Homebrew…", progress, the result in the row, a failure with "Copy Command"; the read-only wording goes.

**Tests, by AC**
- Runtime: the exact CoreSimulator simctl call without `--keep-asset`, never `xcrun` or the shim; one refusal table (unknown id, `all`, a build, `--unusable`, booted, `deletable: false`, `Deleting`, no `runtimeIdentifier`, outside AssetsV2, `AssetsV2Evil`, `..`, devices unknown, first launch pending), with no delete run.
- No `--actions` and no TTY → 2, nothing run; `--yes` → 2; every `ActionList` error refuses the whole list.
- Brew: the area's brew, `cleanup` alone, the three variables, `cleanup -n` with them; refusals (Intel prefix, cache elsewhere, cellar outside); a missing cache folder compared as a path; `cache.homebrew` re-measured and the area stale.
- Journal: one `area-action` line at version 2 per run, none for a dry run; clean lines carry `kind: "item"`; lock held → 75.
- Freed bytes: freed, removed-not-freed, still-deleting after the deadline, and a stop during the wait journaled as still-deleting.
- Swift: a scan queued behind `disk free` waits and is done; two free requests are two runs; the button rules.
- A permission error → failed, needing an administrator, with the exact Terminal command; other failures and a timeout without it.
- Contract: `freeConfirmedActions` defined in `DiskFreeRun` and called only by `AreaActionSheet`; `.diskFree` only in `CLIRunner` and `DiskFreeRun`; nothing scheduled mentions it; `structure.rs` keeps `xcrun`, `--keep-asset`, `--prune` and `remove_` out of `free/`; schema and enum mirrors.
- Schemas checked like the others; app fixtures round-trip.

Fakes only: an extended `FakeCommands` (exe paths, call sequences, failures, a hook that shrinks the fixture asset folder) in core; shell-script fakes under `--system-root` and a temp HOME in the CLI tests; a `disk free` branch in `FakeCLI`. No test runs the real `simctl` or `brew`.

## Acceptance Criteria

- [x] Both ADR amendments are accepted after a critic pass, before any code that runs an action. — evidence: critic (2026-10-28) returned revise with 13 findings, all applied; the re-check approved after one more sentence (a missing Homebrew cache folder), applied verbatim; safety ADR Amendments entry of 2026-10-28, menu bar ADR §1, §6 item 11, §7
- [x] Test (fake commands port): a confirmed list `[{action: simctl-runtime-delete, identifier}]` runs exactly the real `simctl runtime delete <identifier>`, with no `--keep-asset` and never the shim or `xcrun`. An unknown id, `all` or a build, a runtime with a booted device, one with `deletable: false`, one in the state `Deleting`, one without a `runtimeIdentifier`, and one whose `parentMountPath` lies outside AssetsV2 are each refused before any run. — evidence: core `disk_free.rs`: `a_runtime_is_deleted_with_coresimulators_simctl_alone_and_its_bytes_are_measured`, `every_runtime_the_probe_cannot_vouch_for_is_refused_before_any_delete` (booted and booting devices included), `without_ready_developer_tools_nothing_is_listed_or_deleted`; `structure.rs::disk_free_runs_only_its_two_commands` (projectstore:reviewer, 2026-10-28: Met)
- [x] Test: `disk free` without `--actions` and without a TTY exits with the refusal code and runs nothing. `--yes` is not accepted. — evidence: CLI `disk_free_cli.rs`: `without_actions_and_without_a_terminal_it_refuses_and_runs_nothing`, `there_is_no_yes` (projectstore:reviewer, 2026-10-28: Met)
- [x] Test: `brew-cleanup` runs `/opt/homebrew/bin/brew cleanup` with no other arguments and `HOMEBREW_NO_AUTOREMOVE=1` in its environment, and its dry run runs `brew cleanup -n` with the same environment. After the run, the `cache.homebrew` finding in `last-scan.json` is measured again. — evidence: core `brew_cleanup_runs_alone_from_the_areas_brew_with_no_autoremove`, `after_brew_cleanup_the_homebrew_cache_finding_is_measured_again`; CLI `brew_cleanup_runs_alone_with_no_autoremove_and_its_dry_run_changes_nothing`, `after_brew_cleanup_the_last_scan_is_measured_again` (projectstore:reviewer, 2026-10-28: Met)
- [x] Test: every run writes one `actions.jsonl` line of the area-action kind, with the journal's own schema version, the command and the result; the scan report's `SCHEMA_VERSION` is unchanged. A run takes the exclusive lock and reports busy when it is held. — evidence: core `the_journal_has_its_own_version_and_the_scan_reports_is_unchanged`; CLI `a_confirmed_runtime_is_deleted_journaled_and_left_out_of_the_saved_disk_report`, `a_held_lock_exits_75_and_runs_nothing`; `clean_safety.rs:149-152` (projectstore:reviewer, 2026-10-28: Met)
- [x] Test (fake commands port): a runtime that leaves the list while its AssetsV2 folder keeps its size yields "not freed" with the bytes, and a runtime still listed after the timeout yields "still deleting". — evidence: core `a_runtime_that_left_the_list_while_its_folder_kept_its_size_is_removed_not_freed`, `a_runtime_still_listed_at_the_deadline_is_still_deleting`, `a_stop_during_the_wait_ends_it_as_still_deleting_and_journals_it`, `a_stop_during_the_delete_call_reads_the_list_once` (projectstore:reviewer, 2026-10-28: Met)
- [x] Test with a fake CLI: `disk free` is queued in the `CLIRunner`; a scan queued behind it waits and does not fail busy. — evidence: Swift `DiskFreeTests`: `scanWaits`, `neverCoalesced`, `stopDuringRun` (projectstore:reviewer, 2026-10-28: Met)
- [x] Test: a tool exiting with an authorization failure yields a result with the Terminal command. — evidence: core `a_failing_tool_carries_its_terminal_command_and_names_an_administrator_only_on_a_permission_error`; CLI `a_permission_error_fails_with_the_terminal_command_and_exits_1_in_text` (projectstore:reviewer, 2026-10-28: Met)
- [x] Contract test: the sheet's confirm button is the only caller of `disk free` in the app, and the scheduled path never calls it. — evidence: Swift `ContractTests::freeCallers` (the one `dryRun: false` run sits in `freeConfirmedActions`, called only from `AreaActionSheet.swift`; the scheduled files never name it) (projectstore:reviewer, 2026-10-28: Met)
- [x] `docs/disk-free.schema.json` is generated and checked like the other schemas. — evidence: core `committed_free_schema_is_current`; Swift `ContractTests::freeContracts` (projectstore:reviewer, 2026-10-28: Met)
- Manual checks (2) moved to [Owner checks: manual acceptance across stories](../../../ops/owner-checks-manual-acceptance-across-stories.md) on 2026-10-30, by the owner's decision.

## Final Summary

**What changed.** `ddc disk free` runs the owning tool on a whole-disk area: `simctl runtime delete <asset id>` for one simulator runtime, and `/opt/homebrew/bin/brew cleanup` with `HOMEBREW_NO_AUTOREMOVE=1` for Homebrew. Each runs only after its own dry run and an explicit confirmation: a question per action at a terminal, or a list the app's sheet confirmed, passed with `--actions`. There is no `--yes`. The core module `free/` probes, runs and measures; every run takes the exclusive lock and writes one area-action line to `actions.jsonl` (journal schema 2; the scan report's schema is unchanged). The report is `docs/disk-free.schema.json`, version 1. After a run, `last-disk.json` drops a deleted runtime and marks the Homebrew area stale ("before cleanup"), and `last-scan.json` measures `cache.homebrew` again. In the app, a runtime row and the Homebrew row of the Whole disk view get a button that opens a sheet: the tool's dry run, the exact command with Copy Command, what cannot be undone, then Stop, and the result in the row.

**Why.** The two largest areas outside the home folder are owned by tools that know what is safe to remove; DDC runs those tools instead of deleting their folders (safety ADR amendment of 2026-10-28, menu bar ADR §6 item 11, §7).

**Review fixes (projectstore:reviewer, 2026-10-28).** The Homebrew sheet names `/opt/homebrew/Cellar` and `~/Library/Caches/Homebrew` and says that shell-profile `HOMEBREW_*` settings do not reach a run the app starts, while `brew.env` files do. Ctrl-C at a terminal question ends the process before anything runs; the signal handler is installed only once the list is known, and a dry run ended by a signal ends by it too. A stop while a tool runs reports what the list shows (gone, still deleting) or "stopped before the tool finished". Any device state other than Shutdown refuses a runtime delete. The confirm button is disabled during a whole-disk scan, and a scan waits while a sheet waits for its confirmation. A new whole-disk measure clears the rows' result lines.

**Tests executed.** `make check` (fmt, clippy `-D warnings`, all Rust tests), `swift test` (250 tests), and an unsigned `xcodebuild` of the app. No test runs the real `simctl` or `brew`; the CLI tests use a temporary HOME and data directory.

**Risks and follow-ups.**
- Whether Xcode 27's runtime `path` always lies inside its `.asset` folder is settled by the owner's manual check (AC 11).
- Two Whole disk windows would each show the sheet.
- The Homebrew cache folder may be a symlink; the measure follows the area's rules.
- With a terminal on stdout, `--json` without `--actions` prints the questions and then the JSON.
- The journal bump to schema 2 is not described in the README.
- Accepted as notes: the 0600 list file is created then written (a short race in the user's own temp folder); a journal write error mid-run ends the run as in `clean`; a lock wait longer than `NATIVE_TIMEOUT` is not bounded separately.
- ACs 11 and 12 are the owner's; the story stays in progress until they are recorded.

## Dependencies

- "Whole disk: what lies outside the home folder" (DDC-002): the areas and the runtime data
- "Whole disk in the app" (DDC-002): the view the buttons live in
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]: §5, §6 and §7 amended
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]: §6 and §7 amended

## Attachments

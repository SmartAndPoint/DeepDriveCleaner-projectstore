---
type: story
id: "story-whole-disk-free-space-with-the-owning-tool"
epic: "DDC-002"
title: "Whole disk: free space with the owning tool"
status: in-progress
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [cli, app, safety, whole-disk]
code_refs: []
specs: []
started_at: "2026-10-28T15:24:31.635Z"
closed_at: null
plan_updated_at: "2026-10-28T15:24:31.635Z"
---

# Whole disk: free space with the owning tool

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
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
- [ ] **CLI: `ddc disk free --actions FILE|- [--dry-run] [--json]`.**
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
- [ ] **App:**
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
- [ ] Test (fake commands port): a confirmed list `[{action: simctl-runtime-delete, identifier}]` runs exactly the real `simctl runtime delete <identifier>`, with no `--keep-asset` and never the shim or `xcrun`. An unknown id, `all` or a build, a runtime with a booted device, one with `deletable: false`, one in the state `Deleting`, one without a `runtimeIdentifier`, and one whose `parentMountPath` lies outside AssetsV2 are each refused before any run.
- [ ] Test: `disk free` without `--actions` and without a TTY exits with the refusal code and runs nothing. `--yes` is not accepted.
- [ ] Test: `brew-cleanup` runs `/opt/homebrew/bin/brew cleanup` with no other arguments and `HOMEBREW_NO_AUTOREMOVE=1` in its environment, and its dry run runs `brew cleanup -n` with the same environment. After the run, the `cache.homebrew` finding in `last-scan.json` is measured again.
- [ ] Test: every run writes one `actions.jsonl` line of the area-action kind, with the journal's own schema version, the command and the result; the scan report's `SCHEMA_VERSION` is unchanged. A run takes the exclusive lock and reports busy when it is held.
- [ ] Test (fake commands port): a runtime that leaves the list while its AssetsV2 folder keeps its size yields "not freed" with the bytes, and a runtime still listed after the timeout yields "still deleting".
- [ ] Test with a fake CLI: `disk free` is queued in the `CLIRunner`; a scan queued behind it waits and does not fail busy.
- [ ] Test: a tool exiting with an authorization failure yields a result with the Terminal command.
- [ ] Contract test: the sheet's confirm button is the only caller of `disk free` in the app, and the scheduled path never calls it.
- [ ] `docs/disk-free.schema.json` is generated and checked like the other schemas.
- [ ] Manual (owner): deleting an unused simulator runtime from the app frees its size, and `xcrun simctl runtime list` no longer lists it. Whether macOS asks for a password is recorded here.
- [ ] Manual (owner): Homebrew cleanup from the app shows the same list as `HOMEBREW_NO_AUTOREMOVE=1 brew cleanup -n` in Terminal (with no other `HOMEBREW_*` variable set in the shell), then frees about that much.

## Final Summary

## Dependencies

- "Whole disk: what lies outside the home folder" (DDC-002): the areas and the runtime data
- "Whole disk in the app" (DDC-002): the view the buttons live in
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]: §5, §6 and §7 amended
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]: §6 and §7 amended

## Attachments

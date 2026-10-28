---
type: story
id: "story-whole-disk-free-space-with-the-owning-tool"
epic: "DDC-002"
title: "Whole disk: free space with the owning tool"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [cli, app, safety, whole-disk]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
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

- [ ] **Amendment to the safety ADR, "Cleanup safety model: risk classes, plan and confirmation".**
  - **§6.** Replace the `brew cleanup` sentence: `brew cleanup` runs only as the Homebrew area's action, from a confirmation.
  - **The area action, a new kind of native action.** It is not a catalog rule, and it has no path inside the scan roots. Its guard is a location probe:
    - a runtime's `parentMountPath` must lie under `/System/Library/AssetsV2`;
    - `brew --cellar` and `brew --cache` must match the Homebrew area's paths.
  - **§5.** `ddc disk free` takes the exclusive lock, because `brew cleanup` writes inside `~/Library/Caches/Homebrew`, a scan root.
  - **§7.** Every run writes an `actions.jsonl` line with the area, the action, the exact command, the bytes before and after, and the result. An area action has no `path`, `(dev, ino)`, `rule` or `class`, so this is a new line kind with a `kind` discriminator. The journal gets its own `JOURNAL_SCHEMA_VERSION`, as `clean` and `progress` have theirs; today it borrows the scan report's `SCHEMA_VERSION`, and bumping that would invalidate every `last-scan.json`.
  - **Consent.** This is a separate channel, not `--yes`, because §5 keeps `--yes` for the `safe` class without a terminal. The app passes a confirmed action list (`--actions FILE|-`), as `clean --items` does. At a terminal, a prompt asks.
- [ ] **Amendment to the menu bar ADR, "Menu bar app: Swift shell over the ddc CLI".**
  - **§6:** the new commands `ddc disk`, `ddc disk free` and its `--dry-run`, with their schemas and exit codes.
  - **§7:** one caller per destructive call, extended to `disk free`. The caller is the Whole disk sheet's confirm button, and `ContractTests` gains a test that names it.
  - **§1:** `disk free` is a `CLIRunner` queue kind beside `clean`, never in the read-only `.disk` lane of "Whole disk in the app". It takes the ddc lock, so a Scan now during a long `brew cleanup` waits in the queue instead of failing busy. A `disk free` does not wait for a `ddc disk` in the read-only lane; when they overlap, the last to finish writes `last-disk.json`, as "Whole disk: what lies outside the home folder" accepts.
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
    - It runs plain `brew cleanup`, without `-s` or `--prune=all`. `brew` is found through `Env::tool_dirs()`.
    - The dry run is `brew cleanup -n`, and the sheet lists what it would remove and the estimate.
    - It touches mainly three places, and the sheet names them: the Cellar, the Caskroom, and `~/Library/Caches/Homebrew`. It also prunes `~/Library/Logs/Homebrew`, Homebrew's own caches and broken symlinks in its prefix.
    - Started by the app, brew does not see the user's shell profile, such as `HOMEBREW_NO_CLEANUP_FORMULAE`. The sheet says so.
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

## Acceptance Criteria

- [ ] Both ADR amendments are accepted after a critic pass, before any code that runs an action.
- [ ] Test (fake commands port): a confirmed list `[{action: simulator-runtime, identifier}]` runs exactly the real `simctl runtime delete <identifier>`, with no `--keep-asset` and never the shim or `xcrun`. An unknown id, a runtime with a booted device, and a runtime whose `parentMountPath` lies outside AssetsV2 are each refused before any run.
- [ ] Test: `disk free` without `--actions` and without a TTY exits with the refusal code and runs nothing. `--yes` is not accepted.
- [ ] Test: `homebrew-cleanup` runs `brew cleanup` with no other arguments, and its dry run runs `brew cleanup -n`. After the run, the `cache.homebrew` finding in `last-scan.json` is measured again.
- [ ] Test: every run writes one `actions.jsonl` line of the area-action kind, with the journal's own schema version, the command and the result; the scan report's `SCHEMA_VERSION` is unchanged. A run takes the exclusive lock and reports busy when it is held.
- [ ] Test (fake commands port): a runtime that leaves the list while its AssetsV2 folder keeps its size yields "not freed" with the bytes, and a runtime still listed after the timeout yields "still deleting".
- [ ] Test with a fake CLI: `disk free` is queued in the `CLIRunner`; a scan queued behind it waits and does not fail busy.
- [ ] Test: a tool exiting with an authorization failure yields a result with the Terminal command.
- [ ] Contract test: the sheet's confirm button is the only caller of `disk free` in the app, and the scheduled path never calls it.
- [ ] `docs/disk-free.schema.json` is generated and checked like the other schemas.
- [ ] Manual (owner): deleting an unused simulator runtime from the app frees its size, and `xcrun simctl runtime list` no longer lists it. Whether macOS asks for a password is recorded here.
- [ ] Manual (owner): Homebrew cleanup from the app shows the same list as `brew cleanup -n` in Terminal, then frees about that much.

## Final Summary

## Dependencies

- "Whole disk: what lies outside the home folder" (DDC-002): the areas and the runtime data
- "Whole disk in the app" (DDC-002): the view the buttons live in
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]: §5, §6 and §7 amended
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]: §6 and §7 amended

## Attachments

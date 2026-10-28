---
type: story
id: "story-vm-images-docker-prune-runs-only-while-docker-desktop-answers"
epic: "DDC-002"
title: "VM images: Docker prune runs only while Docker Desktop answers"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-29
external_refs: {}
tags: [core, cli, app, native-command, docker, user-report]
code_refs: ["crates/ddc-core/src/class.rs", "crates/ddc-core/src/rules/catalog.rs", "crates/ddc-core/src/rules/mod.rs", "crates/ddc-core/src/env.rs", "crates/ddc-core/src/exec/native.rs", "crates/ddc-core/src/exec/mod.rs", "crates/ddc-core/src/exec/guard.rs", "crates/ddc-core/src/exec/preflight.rs", "crates/ddc-core/src/exec/journal.rs", "crates/ddc-core/src/exec/report.rs", "crates/ddc-core/src/progress.rs", "crates/ddc-cli/src/prompt.rs", "docs/clean.schema.json", "docs/progress.schema.json", "crates/ddc-core/tests/clean_safety.rs", "crates/ddc-core/tests/in_use.rs", "app/Packages/DDCKit/Sources/DDCKit/CleanReport.swift", "app/Packages/DDCKit/Sources/DDCKit/ProgressEvent.swift", "app/Packages/DDCKit/Sources/DDCKit/CleanWording.swift", "app/Packages/DDCKit/Sources/DDCKit/CleanResult.swift", "app/Packages/DDCKit/Tests/DDCKitTests/"]
specs: []
started_at: "2026-10-28T18:28:36.673Z"
closed_at: null
plan_updated_at: "2026-10-28T18:28:36.673Z"
---

# VM images: Docker prune runs only while Docker Desktop answers

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

From a user's screen recording of 2026-10-28 (app 0.4.0). The user selected VM and container images → `~/Library/Containers/com.docker.docker` (38.9 GiB, rule `vm.docker-desktop`, "Run its tool's prune commands"). The confirmation sheet said "No warnings: nothing is predicted to be skipped." Both prune commands then failed, in two cleans in a row:

```
docker --context desktop-linux builder prune -f failed (exit 1)
docker --context desktop-linux image prune -f failed (exit 1)
```

The item stayed selected, the group stayed at 38.9 GiB, and the user read it as "selected but not deleted".

**This breaks the safety ADR.** [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model]] §1, row `native-only`: the prune commands run "only when the daemon is running; otherwise a report line with instructions", and "if that context does not exist, instructions only". The code checks neither. `exec/native.rs` finds `docker` on the path and runs the command; a non-zero exit is a failure, and the executor runs the second command after the first failed (`exec/mod.rs`, the native-only loop). `exec/preflight.rs` predicts only `native-unavailable` (no `docker` binary).

**Why docker failed is not known.** The recording shows only the exit code; docker's own output went to the journal (`CommandResult.output`, the last 2000 characters of stdout and stderr), which the result view does not show. Three hypotheses, each handled below:
1. Docker Desktop was not running, or still starting, so its socket did not answer.
2. The `desktop-linux` context does not exist (an older Docker Desktop, or a `docker` CLI from another runtime).
3. Docker Desktop was uninstalled and its 38.9 GiB folder left behind, with a `docker` CLI still on the path from another runtime. Then the item can never be pruned, and the folder is the user's to delete in Finder: the app never removes a `native-only` folder (§1).

What changes:

- **The preflight stays process-free.** The dry run "runs no native command" (menu bar ADR §6 item 2; the header of `preflight.rs`: native tools are only looked up, never run), and this story keeps that. The preflight predicts from what it can read: no `com.docker.backend` process in the `ProcessTable` it already holds (the safety ADR names that process) → "Docker Desktop is not running"; no `contexts/meta/<sha256 of "desktop-linux">/meta.json` under the docker config folder (`$DOCKER_CONFIG`, else `~/.docker`) → "the desktop-linux context does not exist".
- **The executor probes before it prunes**, right before the first command: `docker --context desktop-linux version --format '{{.Server.Version}}'`, with a 5 s timeout. A failure skips the item; no prune command runs. This is the ADR's "only when the daemon is running", made concrete; the ADR text needs no change, and the story's Final Summary records the probe as its implementation. The rule's `NativeCommand` gains an optional readiness probe beside its `location` probe, so `podman` can get its own later (`podman info`).
- **A new skip reason**, `tool-not-ready { program, detail }`, with `detail` one of "Docker Desktop is not running or not answering", "the desktop-linux context does not exist", or the last non-empty line of docker's output. It is a skip, not a failure: nothing was touched. The wording says what to do: "Start Docker Desktop, then clean again. If you no longer use Docker Desktop, its folder is yours to delete in Finder."
- **A prune that still fails after the probe** shows docker's own last line. The core already reports it in `CommandResult.output`; DDCKit's `CleanReport.CommandResult` does not decode it. It decodes it now, and the result view shows "docker image prune failed: <last line>" instead of "failed (exit 1)".
- **Wording:** `native-unavailable` shrinks to "is not installed"; "or its runtime is not running" moves to the new reason.

**Schema versions.** The Rust core ADR's policy: a new value of `skipped[].reason` bumps the schema version, because strict decoders such as Swift `Codable` reject unknown values. The precedent is the 2026-10-20 skip reasons, which raised the clean report from 1 to 2. Every document that serializes the reason is bumped:
- the clean report, `CLEAN_SCHEMA_VERSION` 2 → 3, `docs/clean.schema.json`, DDCKit `CleanReport.schemaVersion` 3;
- the action journal, `JOURNAL_SCHEMA_VERSION` 2 → 3 (its lines carry the outcome with the reason);
- the progress events, `PROGRESS_SCHEMA_VERSION` 1 → 2 (each item event carries its outcome), `docs/progress.schema.json`, DDCKit `ProgressEvent.schemaVersion` 2.

The app embeds its own CLI, so the app and the CLI move together; the contract tests that pin the versions move with them.

## Decomposition

- [x] Core: `NativeCommand` gains `ready: Option<ReadyProbe>`; the two Docker prune commands get the `version` probe. `exec/native.rs` runs it once before the commands of an item and returns `Native::NotReady(detail)`; `exec/mod.rs` maps it to `SkipReason::ToolNotReady` and runs no command.
- [x] Core: `exec/preflight.rs` predicts `tool-not-ready` from the process table and the context's metadata file, once per check, and runs nothing.
- [x] Core: the three schema versions bumped; schemas regenerated (`DDC_UPDATE_SCHEMA=1 cargo test`).
- [x] CLI: the human `ddc clean` output says "skipped: Docker Desktop is not running or not answering — start it and run again".
- [x] DDCKit: decode the new reason and `CommandResult.output`; the versions; `CleanWording` for the new reason and the narrower `native-unavailable`; the result's failure line uses the output's last line.

## Implementation Plan

Planned by `projectstore:planner` on 2026-10-28. No spec covers the story; the contract is the safety ADR §1 (`native-only`), the menu bar ADR §6 item 2 and the Rust core ADR's versioning policy. Core first, then the CLI, then DDCKit.

1. **Types.** `class.rs`: `ReadyProbe { program, args, process, context_meta, down, known }` beside `LocationProbe`; `NativeCommand.ready: Option<ReadyProbe>`. `rules/catalog.rs`: both `DOCKER_PRUNE` entries get the probe `docker --context desktop-linux version --format {{.Server.Version}}`, process `com.docker.backend`, context file `contexts/meta/fe9c6bd7a66301f49ca9b6a70b217107cd1284598bfc254700c989b916da791e/meta.json` (the sha256 of `desktop-linux`, hard-coded: no new hash dependency); `PODMAN_PRUNE` and `native_env` get `ready: None`.
2. **`DOCKER_CONFIG`.** `env.rs`: `Env::from_process` (the one place allowed to read the environment) stores it; `docker_config()` falls back to `home/.docker`; `with_paths` leaves it unset, so tests never see the developer's environment.
3. **Executor.** `exec/native.rs`: `Native::NotReady(String)` and `ready(probe, commands, opts)` with a 5 s `READY_TIMEOUT` beside `NATIVE_TIMEOUT`: a timeout or empty output gives `down`, a known needle in the last non-empty line gives its detail, otherwise that line itself (a `last_line` helper beside `tail`); a cancel after the probe gives `Cancelled`. `exec/mod.rs`: `SkipReason::ToolNotReady { program, detail }`; in `act`'s `NativeOnly` branch, `commands.find` first (a missing tool stays `native-unavailable`), then the first command's probe once, inside the progress `tool` bracket; `NotReady` → skipped, `Cancelled` → not run; no prune runs in either case.
4. **Preflight, process-free.** `exec/guard.rs` keeps the process names of the snapshot `Guard::new` already takes and answers `running(name)` (no second `processes()` call). `exec/preflight.rs::check`, `NativeOnly` arm, after the `native-unavailable` check: no `com.docker.backend` → `tool-not-ready` "Docker Desktop is not running or not answering"; else no context file under `env.docker_config()` → "the desktop-linux context does not exist". `check` gets `env`. The module header says the probe is the executor's only.
5. **Versions.** `CLEAN_SCHEMA_VERSION` 3, `JOURNAL_SCHEMA_VERSION` 3, `PROGRESS_SCHEMA_VERSION` 2; `DDC_UPDATE_SCHEMA=1 cargo test` regenerates `docs/clean.schema.json` and `docs/progress.schema.json`. Pins to move: `tests/clean_safety.rs` (journal 2), `tests/disk_free.rs` and `ddc-cli/tests/disk_free_cli.rs` (journal 2), `tests/progress.rs` literals (1).
6. **CLI.** `prompt.rs::reason`: `ToolNotReady` → "{detail} — start it and run again"; `NativeUnavailable` → "{program} is not installed".
7. **DDCKit.** `CleanReport.swift`: `case toolNotReady(program:detail:)`, its tag, `detail` key, `CommandResult.output` (`decodeIfPresent ?? ""`), `schemaVersion = 3`; `ProgressEvent.schemaVersion = 2`; `CleanWording.reason` for the new case, `native-unavailable` narrowed to "is not installed"; `CleanResult.make` names a failed command short ("docker image prune") with the output's last non-empty line, "(exit N)" when empty. Fixtures and inline JSON move with the versions: `Fixtures/clean-report.json`, `CleanResultTests`, `GroupCleanTests`, `UserDataTests` (3); `ProgressTests`, `PopoverModelTests`, `DiskModelTests`, `CLIRunnerTests` (2, and their "other version" cases to another number); `ContractTests` tag count 22 and the sample line gains `detail`.
8. **Tests.** `tests/clean_safety.rs` with `FakeCommands`: probe exit 1, the context message, a timeout, a cancel on the probe, a failed prune after a good probe; `native_only_rules_never_delete_or_trash` gets a good probe answer. `tests/in_use.rs` (`predict_and_clean`): no backend process → predicted skip, no command call during the dry run; with the process and the context file → both prunes; `a_runtime_prunes_while_its_daemon_works_inside_its_finding` writes the context file. DDCKit: version 3 decodes the reason and `output`, version 2 is refused.

Pitfalls: clippy forbids `remove_*` in tests and `env::var` outside `Env`; rustdoc `-D warnings`, so paths with `<…>` go in backticks (the doc comments become schema text); `cargo build` before `swift test`, or the real-CLI suites meet a version-2 binary.

## Acceptance Criteria

- [x] Test (core, fake `CommandRunner`): the `version` probe exits 1 → the item is skipped with `tool-not-ready`, neither prune command runs, and the journal line says why. — evidence: `crates/ddc-core/tests/clean_safety.rs::a_docker_desktop_that_does_not_answer_skips_its_item_and_runs_no_prune`
- [x] Test (core): the probe prints docker's "context … does not exist" → the same skip, with that detail. — evidence: `clean_safety.rs::the_probe_names_a_missing_context_or_says_what_docker_said`
- [x] Test (core): the probe succeeds and `image prune` exits 1 with a last line → the item is a failure whose command output carries that line. — evidence: `clean_safety.rs::a_prune_that_fails_after_a_good_probe_carries_dockers_last_line`
- [x] Test (core): the probe's timeout ends as `tool-not-ready`, not as a hang; a cancel during the probe runs no prune command. — evidence: `clean_safety.rs::a_probe_that_times_out_skips_and_a_cancel_during_it_runs_no_prune`
- [x] Test (core): the preflight, with a fake process table that has no `com.docker.backend`, predicts `tool-not-ready` for every Docker item and runs no command (the fake runner records no call); with the process and the context file present, it predicts the prune. — evidence: `crates/ddc-core/tests/in_use.rs::the_dry_run_predicts_a_docker_desktop_that_cannot_answer_without_running_anything`, `in_use.rs::a_runtime_prunes_while_its_daemon_works_inside_its_finding`
- [x] Test (DDCKit): `CleanReport` at version 3 decodes `tool-not-ready` and a command's `output`; version 2 is refused; the strict list of reason tags includes the new one; the result's failure line shows the output's last line. — evidence: `CleanResultTests` "Docker Desktop not answering is a skip that says why…", `ContractTests.schemaEnums`, `ContractTests.skipReasonsHaveText`; `swift test` 251 tests green
- [x] Contract tests: the three versions agree between Rust and DDCKit. — evidence: `ContractTests.cleanContracts`, `ContractTests.progressContracts`
- [x] `make gate` green. — evidence: pre-push `make gate` passed on push of code commit 5cb3a28 (nextest 428/428, swift tests, clippy, doc)
- [ ] Manual (owner): with Docker Desktop quit, select the Docker item: the sheet says it will be skipped because Docker Desktop is not running. Start it, Check Again, clean: both prune commands run and the result says what they reported.

## Final Summary

**What changed (code commit 5cb3a28).** Each Docker prune command now carries a readiness probe (`ReadyProbe` on `NativeCommand.ready`): `docker --context desktop-linux version --format {{.Server.Version}}`, run once per item with a 5 s timeout before any prune. A probe that fails, times out or prints nothing ends the item as `skipped` with the new reason `tool-not-ready { program, detail }`, and no prune command runs. The detail says why: "Docker Desktop is not running or not answering", "the desktop-linux context does not exist", or docker's own last line. A cancel during the probe ends the item `not-run`. The dry run predicts the same skip without running anything, from the process table (`com.docker.backend`) and the context's metadata file under `DOCKER_CONFIG` or `~/.docker`. A prune that fails after a good probe is a failure whose command `output` carries docker's last line. The clean report, journal and progress schema versions went up (clean report 3); DDCKit decodes the new reason and `CommandResult.output`, and the result's failure line shows the short command name and that last line. `native-unavailable` is now worded "is not installed". The terminal `ddc clean` adds "— start it and run again" only to the not-running detail.

**Why.** In the user's recording of 2026-10-28, both prunes failed with exit 1 after a sheet that said "No warnings": Docker Desktop was not answering, and nothing said so.

**Tests executed.** `make gate-fast` and the pre-push `make gate`: nextest 428/428, `swift test` 251 tests in 29 suites, clippy, fmt, rustdoc. The reviewer (`projectstore:reviewer`) found no blockers. Its should-fix on the CLI wording was applied before the commit.

**Deferred.** The app's advice for a not-ready tool ("Start Docker Desktop, then Check Again.", what to do about a missing context, and "If you no longer use Docker Desktop, its folder is yours to delete in Finder.") belongs to "Review window: Clean is not offered when every item would be skipped" (DDC-002), which owns the sheet's wording. Nits left: `ReadyProbe.context_meta` could be optional for tools without a context; `CleanResult` drops a positional argument that follows a bare `--flag` when it shortens a command; there is no test of the `DOCKER_CONFIG` override.

**Open.** The manual criterion is the owner's, so the story stays in progress until it is checked.

## Dependencies

- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model]] §1, `native-only`: the rule this story makes the code follow
- [[adr/rust-core-library-and-cli-with-json-output|Rust core: library + CLI with JSON output]]: the schema versioning policy
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]] §6 item 2: the dry run runs no native command
- "Release 0.2.1: the Review window in the Dock, and skip what a running process holds open" (DDC-002): `native-only` items are exempt from the in-use check
- "Review window: Clean is not offered when every item would be skipped" (DDC-002): the sheet that shows the prediction
- "Review window: after a clean, only skips that can clear stay selected" (DDC-002)

## Attachments

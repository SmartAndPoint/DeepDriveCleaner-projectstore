---
type: story
id: "story-vm-images-docker-prune-runs-only-while-docker-desktop-answers"
epic: "DDC-002"
title: "VM images: Docker prune runs only while Docker Desktop answers"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [core, cli, app, native-command, docker, user-report]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# VM images: Docker prune runs only while Docker Desktop answers

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
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

- [ ] Core: `NativeCommand` gains `ready: Option<ReadyProbe>`; the two Docker prune commands get the `version` probe. `exec/native.rs` runs it once before the commands of an item and returns `Native::NotReady(detail)`; `exec/mod.rs` maps it to `SkipReason::ToolNotReady` and runs no command.
- [ ] Core: `exec/preflight.rs` predicts `tool-not-ready` from the process table and the context's metadata file, once per check, and runs nothing.
- [ ] Core: the three schema versions bumped; schemas regenerated (`DDC_UPDATE_SCHEMA=1 cargo test`).
- [ ] CLI: the human `ddc clean` output says "skipped: Docker Desktop is not running or not answering — start it and run again".
- [ ] DDCKit: decode the new reason and `CommandResult.output`; the versions; `CleanWording` for the new reason and the narrower `native-unavailable`; the result's failure line uses the output's last line.

## Implementation Plan

## Acceptance Criteria

- [ ] Test (core, fake `CommandRunner`): the `version` probe exits 1 → the item is skipped with `tool-not-ready`, neither prune command runs, and the journal line says why.
- [ ] Test (core): the probe prints docker's "context … does not exist" → the same skip, with that detail.
- [ ] Test (core): the probe succeeds and `image prune` exits 1 with a last line → the item is a failure whose command output carries that line.
- [ ] Test (core): the probe's timeout ends as `tool-not-ready`, not as a hang; a cancel during the probe runs no prune command.
- [ ] Test (core): the preflight, with a fake process table that has no `com.docker.backend`, predicts `tool-not-ready` for every Docker item and runs no command (the fake runner records no call); with the process and the context file present, it predicts the prune.
- [ ] Test (DDCKit): `CleanReport` at version 3 decodes `tool-not-ready` and a command's `output`; version 2 is refused; the strict list of reason tags includes the new one; the result's failure line shows the output's last line.
- [ ] Contract tests: the three versions agree between Rust and DDCKit.
- [ ] `make gate` green.
- [ ] Manual (owner): with Docker Desktop quit, select the Docker item: the sheet says it will be skipped because Docker Desktop is not running. Start it, Check Again, clean: both prune commands run and the result says what they reported.

## Final Summary

## Dependencies

- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model]] §1, `native-only`: the rule this story makes the code follow
- [[adr/rust-core-library-and-cli-with-json-output|Rust core: library + CLI with JSON output]]: the schema versioning policy
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]] §6 item 2: the dry run runs no native command
- "Release 0.2.1: the Review window in the Dock, and skip what a running process holds open" (DDC-002): `native-only` items are exempt from the in-use check
- "Review window: Clean is not offered when every item would be skipped" (DDC-002): the sheet that shows the prediction
- "Review window: after a clean, only skips that can clear stay selected" (DDC-002)

## Attachments

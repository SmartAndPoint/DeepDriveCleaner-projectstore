---
type: story
id: "story-busy-name-the-scan-or-clean-that-holds-the-lock-and-offer-to-stop-it"
epic: "DDC-002"
title: "Busy: name the scan or clean that holds the lock, and offer to stop it"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-02
updated: 2026-11-08
external_refs: {}
tags: [app, cli, lock, clean]
code_refs:
  - crates/ddc-core/src/holders.rs
  - crates/ddc-core/src/store.rs
  - crates/ddc-core/src/refusal.rs
  - crates/ddc-core/src/lib.rs
  - crates/ddc-core/src/platform/mod.rs
  - crates/ddc-core/src/platform/macos/procs.rs
  - crates/ddc-core/src/platform/macos/signals.rs
  - crates/ddc-core/src/platform/linux/procs.rs
  - crates/ddc-core/src/platform/linux/signals.rs
  - crates/ddc-core/src/platform/windows/procs.rs
  - crates/ddc-core/src/platform/windows/signals.rs
  - crates/ddc-cli/src/cmd/lock.rs
  - crates/ddc-cli/src/cli.rs
  - crates/ddc-cli/src/cmd/mod.rs
  - crates/ddc-cli/src/main.rs
  - crates/ddc-cli/src/gate.rs
  - crates/ddc-cli/src/render.rs
  - crates/ddc-cli/src/progress.rs
  - crates/ddc-cli/src/cmd/scan.rs
  - crates/ddc-cli/src/cmd/background.rs
  - crates/ddc-cli/src/cmd/clean.rs
  - crates/ddc-cli/src/cmd/clean_items.rs
  - crates/ddc-cli/src/cmd/disk_free.rs
  - crates/ddc-cli/src/cmd/keep.rs
  - crates/ddc-cli/src/cmd/disk.rs
  - crates/ddc-cli/src/cmd/agent_status.rs
  - docs/lock.schema.json
  - docs/error.schema.json
  - README.md
  - app/Packages/DDCKit/Sources/DDCKit/LockHolders.swift
  - app/Packages/DDCKit/Sources/DDCKit/CLIRunner.swift
  - app/Packages/DDCKit/Sources/DDCKit/RunnerActivity.swift
  - app/Packages/DDCKit/Sources/DDCKit/CleanRun.swift
  - app/Packages/DDCKit/Sources/DDCKit/ReviewModel.swift
  - app/Packages/DDCKit/Sources/DDCKit/PopoverModel.swift
  - app/DeepDriveCleaner/ConfirmSheet.swift
  - app/DeepDriveCleaner/PopoverView.swift
  - app/DeepDriveCleaner/LockBusyPanel.swift
specs: []
started_at: "2026-11-08T08:03:33.314Z"
closed_at: "2026-11-08T11:15:33.417Z"
plan_updated_at: "2026-11-08T08:03:33.314Z"
---

# Busy: name the scan or clean that holds the lock, and offer to stop it

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner, 2026-11-02, on a screenshot: a clean from the Review window ended with "The clean did not finish. Another scan or clean holds the lock. Try again when it ends. Nothing was touched." It was right: a terminal `ddc clean --items` was deleting 200 GiB and held the lock. But the person could not tell what held it, for how long, or what to do.

The owner's rule: **show at once that another process holds the lock, which one, and offer a button to stop it; when no process holds it, show nothing.**

- **Who holds it: one record per holder.** The lock has nine callers in six files. Shared: `scan`, the background scan, `clean --items --dry-run`, and `disk free --dry-run` and its preview. Exclusive: `clean` (its plain `--dry-run` too, which takes the lock before it branches), `clean --items`, `disk free` and `keep remove`. So there can be several shared holders at once, and an exclusive one holds it alone. In production no process holds two locks at once (`disk free` releases its preview's shared lock before it takes the exclusive one), but a test can hold two shared ones, so the record is counted per process.
  - Each holder writes its own record, `<data-dir>/lock.holders/<pid>-<start_us>.json`, right after it takes the lock. The record holds the command, the mode (shared or exclusive), the start time, the origin and, for a clean, its phase and progress. The origin is `app`, `terminal` or `other`. It is a guess: `terminal` means stdin or stderr is a TTY, so the app says "started from a terminal", not "in Terminal". The phase is `preparing` until the first item runs, and `cleaning` after that.
  - The holder removes its record before it releases the lock, and also explicitly before it re-raises a fatal signal, because `signals::reraise` skips `Drop`.
  - A record is stale when its pid is gone, or when that pid's start time differs from the record's by more than two seconds, which means the pid was reused. A stale record is ignored, and the next holder sweeps it. Stale records are normal after a crash or a SIGKILL.
  - The executable is not part of the staleness test, so a holder updated in place (`brew upgrade`, Sparkle) keeps its record. Only `lock stop` checks the executable (below).
  - A holder from a ddc older than this story writes no record. During the move to the new version, such a holder shows as today's line.
- **A read verb.** `ddc lock status [--json]` lists the live holders. It never probes the lock to find them: an exclusive probe would make a concurrent scan fail busy for that instant.
- **Busy as a JSON error.** A command that meets a held lock names the holders on stderr. With `--json`, it prints an error envelope with code `busy` and the holders on stdout, and exits 75 as before. `busy` is a new `ErrorCode` value, so the error schema goes from version 2 to 3. This is the "later change" the platform seam ADR §7 deferred: 75 joins 69 and `root-not-owned` in the envelope. The background scan still skips on a held lock, logs "skipped" and exits 0.
- **In the app.** Three places can meet the lock: the clean sheet's check (its dry run, which runs as the sheet opens), the sheet's Clean, and Scan now in the popover. Each one lists who holds the lock, for example "A clean started from a terminal 3 min ago (pid 4101), deleting item 4 of 8". It offers **Stop** for every holder it names, and one **Wait**. The app shows this as soon as the sheet opens, because the check runs then.
  - **Wait** is new. The app polls `ddc lock status` every few seconds. Once nothing blocking is left, it runs what met the lock again. For a clean, nothing blocking means no holder at all; for the check and for a scan, it means no exclusive holder.
    - For the check, Wait runs the check again. For Clean, it calls the same single entry point that Clean does, so its guards are checked again at the moment it starts. The dry run is not repeated before that clean: today the sheet can stay open for many minutes before Clean too, and the CLI re-checks every item when it runs (safety ADR §5).
    - Wait is offered even when no holder is named, because it does not need a name. Then it has nothing to poll, so it tries the run again every 15 seconds.
    - If a run that Wait started meets the lock again (a terminal took it in between, or its holder writes no record), the app goes back to the busy state with Wait still on, and starts again no sooner than 15 seconds later.
    - A run that Wait started and that fails in any other way (an error, a refusal) is not retried: it shows as that run's usual result.
    - Waiting ends when the person closes the sheet, dismisses the popover's notice, or presses Stop Waiting.
  - **Stop** runs `ddc lock stop --pid P --started T`, where `T` is the record's `start_us`, the integer in its file name. That sends SIGINT only if every check holds: that exact holder still holds the lock, its pid is above 1, it belongs to the same user, and its executable is ddc's binary (the file `deepdrivecleaner`, with Linux's ` (deleted)` suffix ignored). It never sends SIGKILL. When a check fails, `lock stop` refuses and says why. The app then reads `lock status` again and shows the new state.
    - A sent Stop means only that the signal went out. The holder may still be finishing its current item, which can take minutes on a large tree. The app shows "Stopping: finishing the current item…" for that holder and turns its Stop off. A second SIGINT would do no harm, but it would add nothing either.
    - Stop does not start anything by itself. Once the lock is free, the sheet offers Clean again, unless Wait is on.
- **What Stop does.** It is Ctrl-C at the holder's terminal.
  - A clean in `cleaning` finishes its current delete or Trash item and does not run the rest. A tool's command (`brew cleanup`, `go clean -modcache`) is interrupted together with its process group and reported failed. A simulator runtime's deletion continues inside CoreSimulator.
  - `disk free` has its handler from before the lock. Stopped before its first action, it runs none and ends by the signal, and its record is removed. Stopped during an action, it ends after that action's command is interrupted.
  - A clean still in `preparing` (re-checking the items, before its first item) ends at once, and nothing has been removed yet. So do `clean --dry-run`, `clean --items --dry-run`, `keep remove` and a scan. A scan's `last-scan.json` is only ever replaced atomically. None of these has a handler yet, so a holder killed this way leaves a stale record, which the next holder sweeps.
  - The app says which case applies before Stop: "Stop ends it after this item" or "Stop ends it at once".
- **No holder, no button.** When the lock is free, or every record is stale, nothing is shown. A busy result with no live record shows today's line, with Wait but without Stop: the app cannot stop what it cannot name.
- **Out of scope.** `disk free` and Remove Protection keep today's busy line in the app. The CLI already names the holders for them, and their sheets can adopt the same view later.
- **ADRs.** All amendments are dated 2026-11-08.
  - The menu bar ADR gets amendments in four places. §6 item 4: Stop of a process the app did not start. §6 item 5: the envelope on 75. A new §6 item 16: `ddc lock`. §1: the two new queue kinds. §7: Wait is consent given while the sheet is open.
  - The platform seam ADR §7 moves busy into the envelope. Its §3 gates `ddc lock` on `in-use-check`, because both verbs inspect other processes (`lock stop` also reads their executable). Windows declares no capabilities yet, so `lock status` and `lock stop` are refused there, and a busy error there names no holders.
  - The safety ADR §5 gets a note: Stop is SIGINT through the CLI only, never SIGKILL, same user, and it adds no removal path.

## Decomposition

- [x] Vault: the story's corrections, the plan, and the three ADR amendments; `projectstore:critic`
- [x] Core: `holders.rs`, which writes, forgets, reads with an injectable probe, sweeps by pid and start time, and holds `end_by_signal`; `platform::procs::executable` and `platform::signals::interrupt` on each OS; `store::try_lock(env, mode, command)` writes the record, `DataLock`'s `Drop` removes it before unlocking, and `LockError::Busy` carries the holders; all nine callers; a clean's phase and progress in its record, the progress written at most once a second
- [x] CLI: `ddc lock status [--json]` and `docs/lock.schema.json`; `ErrorCode::Busy`, error schema 3 and the envelope on 75; busy text names the holders; then `ddc lock stop --pid --started`; gating on `in-use-check`; README; the Windows native skip list
- [x] DDCKit: decoding `LockStatus` and the busy envelope, the rule for what blocks, the wording, the Wait loop with its 15-second retry, the runner kinds `lockStatus` and `lockStop`
- [x] App: the sheet's check, its Clean and the popover's Scan now show the holders, with Stop, Wait and the stopping state; the single-caller contract test extended to Wait; the owner check in the runbook
- [x] Tests (each PR carries its own)

## Implementation Plan

Planner run 2026-11-08 against `origin/main` d72185c, and critic notes applied the same day. There are five code PRs after this vault PR. Each merges on its own and carries its own unit tests. No PR changes CLI output, or adds a visible command, before the PR that documents it.

1. **Core: holder records and the nine callers.**
   - New `crates/ddc-core/src/holders.rs`.
     - The record: `Holder { pid, start_us, started, command: HolderCommand, mode: LockMode, origin: Origin, phase: Option<Phase>, progress: Option<Progress> }`, with a schemars derive.
     - `HolderCommand` is one of `scan`, `background-scan`, `clean`, `clean-dry-run`, `clean-items`, `clean-items-dry-run`, `disk-free`, `disk-free-dry-run`, `keep-remove`. `Phase` is `preparing` or `cleaning`, and only a clean has one.
     - Liveness is a pure function behind an injectable probe, in the shape of `agents::claude`'s `live`: the probe takes a pid and returns its start time and uid. A record is live when the probe answers, the uid is the caller's, and the start time matches within ±2 s. On Linux the start time is boot time plus ticks, and boot time can move by a second, so the tolerance is two seconds. The executable is not part of this check.
     - Readers list `lock.holders/` through `stats::read_dir` (a structure test requires it), skip names that start with a dot (`write_atomic`'s temporary files) and never remove anything.
     - Only a holder sweeps, after it has taken the lock, and it removes only the records that are not live.
     - A process-wide registry holds this process's record with a count. The first lock writes the record and the last `Drop` forgets it, so two shared locks in one process (`tests/report.rs`'s `exclusive_lock_blocks_a_scan`) stay correct. `end_by_signal` forgets it whatever the count.
   - Two new platform functions, both pinned in `platform/mod.rs` next to `process_start` and `reraise`:
     - `platform::procs::executable(pid)`: macOS makes `pidpath` public, Linux reads `/proc/<pid>/exe`, and Windows returns `None` for now.
     - `platform::signals::interrupt(pid)`: it refuses `pid <= 1`, as `signal_group` refuses `group <= 0`, because `kill(-1, SIGINT)` would reach every process of the user. Otherwise it sends `kill(pid, SIGINT)` on unix and returns `Unsupported` on Windows.
   - `store::try_lock(env, mode, command)`. After the lock is taken, it writes the record through `write_atomic` and sweeps. `DataLock` gets a `Drop` that forgets the record first and only then closes the file, so a record never outlives its lock. On `WouldBlock`, `LockError::Busy { path, holders }` reads the holders: a read only, no sweep. Its `Display` stays "another ddc scan or clean is running (lock …)", so CLI output does not change in this PR.
   - `background.rs` keeps its skip on `Busy { .. }`, and `main.rs` matches the new shape.
   - `holders::end_by_signal(sig) -> !` forgets this process's record, whatever its count, and then calls `platform::signals::reraise`. It runs on the normal thread, not in a handler. It replaces `reraise` in `clean.rs`, `clean_items.rs` and `disk_free.rs`, and also in `agent_status.rs` and `disk.rs`, which hold no lock, so there is one rule. A structure test asserts that `signals::reraise` is called only from `holders.rs`.
   - A clean's phase and progress. `DataLock::set_phase` and `DataLock::set_progress(done, total)` are fed from the exec events through a tee next to `JsonEvents`, so this also works with `--progress` off. Progress writes are throttled to one a second, plus the last one: `write_atomic` fsyncs, and a clean can have thousands of items.
   - Repo rules. Removing a record uses `#[allow(clippy::disallowed_methods)]` with a reason, as `write_atomic` does. No name contains `remove_`, because the `disk_free` structure test refuses that substring. There is no `libc` and no `cfg(unix)` outside `platform/`.
   - Tests in `crates/ddc-core/tests/holders.rs`, on synthetic records with an injected probe:
     - two shared records are both listed, and forgetting one leaves the other;
     - an exclusive record is listed alone;
     - a record with a dead pid or a different start time is ignored by readers and swept by a holder;
     - a record whose start time is within two seconds stays;
     - readers skip dot-files and never remove anything.
   - Tests in `interrupt_cli.rs`:
     - a clean stopped by SIGINT or SIGTERM after k items, through the `Watched` harness, leaves no record;
     - a SIGKILLed scan leaves one, and the next holder sweeps it.
   - The CLI tests that hold the lock from the test binary write a record under the test's own pid. That is harmless, because it is live while the test runs and is forgotten on `Drop`.
   - `scan_cli.rs`'s data-dir listing gains `lock.holders`.
2. **CLI: `lock status` and busy as an error.**
   - New `cmd/lock.rs`, wired as `Command::Lock { Status }` in `cli.rs` and `cmd/mod.rs`, and added to `prints_json`.
   - New `docs/lock.schema.json`: `LockStatus { schema_version: 1, holders }`.
   - `ErrorCode::Busy` and `ErrorBody.holders: Option<Vec<Holder>>`. `ERROR_SCHEMA_VERSION` goes to 3, `refusal.rs`'s doc comment lists the step, and `docs/error.schema.json` is regenerated.
   - Busy in `main.rs`. stderr gets the `ddc:` line plus one line per holder from `render.rs`, the same text `lock status` prints. Under `--json`, stdout gets the envelope. The exit is 75. A streaming reader of `clean --items --dry-run --json` (the app's `CleanRun`) already skips a line it cannot read, so the envelope does it no harm.
   - Gating. `gate::refusal_for` maps `Command::Lock` to `Capability::InUseCheck`. That adds cases to `EVERY`, to `unported_cli.rs`'s `REFUSED` and to a unit test.
   - A structure test asserts there is no `try_lock` in `cmd/lock.rs`.
   - README "## The command line" documents `lock status`, and its exit-75 lines are updated.
   - The `in-use-check` skips go in `.config/native-skips/windows.txt`.
   - Fixed numbers that move: `tests/platform.rs` and `gate.rs`'s `error_schema_version == 2`.
   - Tests in `lock_cli.rs`:
     - two real `deepdrivecleaner scan` runs held under SIGSTOP are both listed by `lock status --json`, which validates through `contract::holds::<LockStatus>`;
     - when one is resumed and ends, the other is still listed;
     - a second clean exits 75 with the holder in its text, and under `--json` with the `busy` envelope.
   - Goldens must not move. Run the whole `scripts/native-linux.sh` before the push.
3. **CLI: `lock stop`.**
   - `ddc lock stop --pid P --started T`, added to `Command::Lock` here, with its README lines.
   - It refuses under root, as `clean` does. It refuses unless all of these hold:
     - the record `P-T` exists and is live;
     - the uid is the caller's;
     - the executable's file name, with ` (deleted)` stripped, is `deepdrivecleaner`.
   - Then it calls `signals::interrupt(P)`. Every refusal exits 2 with its reason.
   - A structure test asserts there is no `SIGKILL`, `kill_group` or `signal_group` in `cmd/lock.rs` or `holders.rs`.
   - Tests:
     - each refusal: a stale record, a different start time, another uid (through the probe), and a pid that is not ddc (the test's own);
     - a clean held under SIGSTOP after k items, then stopped with `lock stop` and resumed, finishes its current item and reports the rest as not run.
4. **DDCKit, no UI.**
   - New `LockHolders.swift`:
     - strict decoders for `LockStatus` and the busy envelope;
     - what blocks a run, for a `Purpose` of `.cleaning`, `.checking` or `.scanning`. These names avoid a `.clean` token, which `ContractTests.cleanHasOneOwner` allows only in `CLIRunner.swift` and `CleanRun.swift`;
     - the wording, by command, phase and origin;
     - the Wait loop, with an injectable clock and interval and the 15-second retry.
   - `CLIRunner` gets two queue kinds, `.lockStatus` and `.lockStop`, with no lane: they take no lock and finish in milliseconds. Their default implementations go in a `CLIRunning` extension, so the fakes keep compiling.
   - `ContractTests` pins `ERROR_SCHEMA_VERSION`, `LOCK_SCHEMA_VERSION`, `HolderCommand`, `LockMode`, `Origin` and `Phase` against the schemas. The fixtures `lock-status.json` and `error-busy.json` use synthetic values only.
   - Tests in `LockHoldersTests` and `CLIRunnerTests`.
5. **App UI.**
   - Busy handling.
     - `CleanRun`'s check maps 75 to the busy envelope's holders, where today it maps it to a fixed `Warnings.busy`.
     - `CleanRun.runClean` branches on `.busy` before it decodes a `CleanReport`.
     - `ReviewModel` gets a busy stage with the holders, the waiting flag and the stopping set. `PopoverModel` gets a structured `busy` next to its notice.
   - The texts go in `CleanWording`. The views are `ConfirmSheet.swift` and `PopoverView.swift`.
   - Stop runs `lock stop` with the pid and `start_us` shown, marks that holder as stopping, and reads `lock status` again.
   - Wait on the check reruns the check. Wait on Clean calls `waitForLockThenClean()`, which calls `cleanConfirmedItems()` once the lock is free. `ContractTests.cleanHasOneOwner` is extended: exactly one call inside `waitForLockThenClean`, and `waitForLockThenClean(` called only from `ConfirmSheet.swift`. A callers test pins `lock stop` to one file.
   - Tests in `CleanRunTests` and `PopoverModelTests`:
     - holders shown with Stop and Wait, in the check, in Clean and in Scan now;
     - with no holders, today's line with Wait and no Stop;
     - Wait starts the run once nothing blocks;
     - a run that meets 75 again goes back to busy and retries no sooner than 15 s later;
     - Stop passes the shown pid and `start_us`, and turns that holder to stopping;
     - closing the sheet ends the wait.
   - The Debug xcodebuild of the app target, then `make ddckit-test`.
   - The runbook "Owner checks: manual acceptance across stories" gets a section.

Code refs, proposed: `crates/ddc-core/src/holders.rs`, `crates/ddc-core/src/store.rs`, `crates/ddc-core/src/refusal.rs`, `crates/ddc-core/src/platform/`, `crates/ddc-cli/src/cmd/lock.rs`, `crates/ddc-cli/src/main.rs`, `crates/ddc-cli/src/gate.rs`, `crates/ddc-cli/src/render.rs`, `crates/ddc-cli/src/progress.rs`, `docs/lock.schema.json`, `docs/error.schema.json`, `app/Packages/DDCKit/Sources/DDCKit/LockHolders.swift`, `app/Packages/DDCKit/Sources/DDCKit/CLIRunner.swift`, `app/Packages/DDCKit/Sources/DDCKit/CleanRun.swift`, `app/Packages/DDCKit/Sources/DDCKit/ReviewModel.swift`, `app/Packages/DDCKit/Sources/DDCKit/PopoverModel.swift`, `app/DeepDriveCleaner/ConfirmSheet.swift`, `app/DeepDriveCleaner/PopoverView.swift`. The epic DDC-002 footprint gains `docs/lock.schema.json` and `docs/error.schema.json`.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [x] Test (core, injected probe): two shared holder records are both listed by `holders()`, and forgetting one leaves the other; an exclusive one lists alone. Test (CLI): two real `deepdrivecleaner scan` runs are both listed by `lock status`, and when one ends the other stays — evidence: `crates/ddc-core/tests/holders.rs` `two_shared_records_are_both_listed_and_forgetting_one_leaves_the_other`, `an_exclusive_record_is_listed_alone_with_its_mode`; `crates/ddc-cli/tests/lock_cli.rs` `two_running_scans_are_both_listed_and_the_one_left_stays_when_the_other_ends` (#133, #138)
- [x] Test (core): a record whose pid is gone or whose start time differs by more than two seconds is ignored and swept by the next holder, and one within two seconds stays; a clean stopped by SIGINT during its items leaves no record (removed before `reraise`) — evidence: `crates/ddc-core/tests/holders.rs` `a_record_is_stale_when_its_pid_is_gone_or_started_at_another_time`, `the_next_holder_sweeps_stale_records_and_keeps_live_ones`; `crates/ddc-cli/tests/interrupt_cli.rs` `a_clean_interrupted_after_its_kth_item_ends_the_current_one_and_leaves_no_record` (#133)
- [x] Test (CLI): `ddc lock status --json` lists live holders and validates against its schema; a second `ddc clean` while one runs exits 75 with the holders in text, and with `--json` an error report with code `busy` (error schema version 3) — evidence: `crates/ddc-cli/tests/lock_cli.rs` (`contract::holds("lock")`), `a_clean_that_meets_a_held_lock_names_the_holder`; `clean_cli.rs` `scan_does_not_start_while_a_clean_holds_the_lock`; `ERROR_SCHEMA_VERSION` 3 in `crates/ddc-core/src/refusal.rs` (#138)
- [x] Test (CLI): `ddc lock stop --pid P --started T` signals SIGINT to that holder only; it refuses a stale record, a holder whose start time differs, another user's process and a non-ddc pid; it never sends SIGKILL; `interrupt` refuses a pid of 1 or less; it is refused with `capability-absent` where `in-use-check` is not declared (a gate unit test, as Windows declares none) — evidence: `crates/ddc-cli/tests/lock_cli.rs` `lock_stop_sends_nothing_without_a_live_record_or_to_a_process_that_is_not_ddc`, `lock_stop_interrupts_a_running_clean_after_its_current_item`; `holders.rs` units `only_a_live_holder_that_runs_ddc_may_be_stopped`; `tests/holders.rs` `interrupt_refuses_a_pid_of_one_or_less`; `structure.rs` `lock_stop_never_kills_or_signals_a_group`; `gate.rs` test `a_command_whose_capability_is_absent_is_refused_with_its_name` (#139)
- [x] Test (DDCKit): the busy state with holders shows them with Stop and Wait in the sheet's check, its Clean and Scan now; with none it shows today's line with Wait and no Stop; Wait starts the run once `lock status` shows no blocking holder, and a run that meets 75 again goes back to busy and retries no sooner than 15 seconds later; Stop calls `lock stop` with the shown pid and `start_us` and shows the holder as stopping — evidence: `LockHoldersTests` (decoders, blocking, wording, Stop, Wait), `CleanRunTests` `checkBusyNamesHolders`, `cleanBusyThenWaitCleans`, `cleanBusyUnnamedRetries`, `stopHolder`, `stopRefused`, `stopWaitingStartsNothing`, `waitFlagIsSpent`, `checkWaitsAgain`, `closingEndsWait`; `PopoverLockTests`; `ContractTests.lockContracts` and `cleanHasOneOwner` (#140, #141)
- [x] A clean stopped this way during its items finishes its current delete or Trash item and reports the rest as not run (existing cancellation tests keep passing) — evidence: `crates/ddc-cli/tests/lock_cli.rs` `lock_stop_interrupts_a_running_clean_after_its_current_item`; `interrupt_cli.rs` `a_clean_stopped_after_its_kth_item_ends_the_current_one_and_runs_no_other` still passes (#139)
- [x] `make check` and `make ddckit-test` pass — evidence: `make gate-fast` in each Rust PR's pre-push and its CI Gate (#133, #137, #138, #139); `make ddckit-test` 576 tests (#141), three runs in a row; the Windows native leg 1069/1069 and the Linux unit tier 1033/1033 on the owner's machines. Main's full `make gate` is red on `script-test` from another story's commit (5170c32, `app/settings.json` in `clean_safety.rs`), not from this one
- [ ] Owner check (runbook "Owner checks: manual acceptance across stories"): with a terminal `ddc clean --items` running, opening the Review window's sheet names it at once, and Stop ends it after the current item

## Final Summary

**What changed.** Five PRs.
- #133: each holder of the data-dir lock writes `<data-dir>/lock.holders/<pid>-<start_us>.json`, forgets it on release and before it ends by a signal (`holders::end_by_signal`, the one caller of `reraise` outside the platform modules), and the next holder sweeps stale ones. `try_lock` takes a `HolderCommand` (nine callers); a clean's phase and progress go to its record. #137 fixed two follow-ups: the record is written after the progress line (an ordering test failed sometimes), and two doc links.
- #138: `ddc lock status [--json]` (`docs/lock.schema.json`), and exit 75 names the holders on stderr and, under `--json`, prints the error envelope with code `busy` (error schema 3).
- #139: `ddc lock stop --pid --started`: one SIGINT, only to a live holder of the same user that runs ddc's binary, with the start read again right before the signal; every refusal exits 2.
- #140: DDCKit's `LockHolders.swift` (decoders, the blocking rule, wording, Stop, the Wait loop) and two queued runner kinds.
- #141: the sheet's check and Clean, and the popover's Scan now, show the holders with Stop and Wait.

**Why.** The owner, 2026-11-02: a clean from the Review window failed busy while a terminal clean held the lock, and the app could not say what held it or offer anything.

**Tests.** Core `tests/holders.rs`; CLI `lock_cli.rs`, `interrupt_cli.rs` (record checks, a SIGINT variant), the busy envelope in `clean_cli`, `keep_cli`, `disk_free_cli`, gate and unported lists, `cases.json`, structure tests; DDCKit `LockHoldersTests`, `CleanRunTests`, `PopoverLockTests`, `ContractTests`. Linux unit tier and the Windows native leg on the owner's machines for each Rust PR.

**Where it differs from the plan.** The phase is set by the executor's `cleaning` event, not by a lock method; the progress is written at most once a second and a flusher writes a held-back change. `lock status` and `lock stop` tests are `cfg(unix)`, so no Windows skips were needed. `ticket(ofPid:)` was dropped: the app's own runs never meet the lock. A change to the list ends Wait, as does a check.

**Risks and follow-ups.** The owner check below is still open. `disk free` and Remove Protection keep the old busy line. A ddc older than this story writes no record, so during an upgrade a busy lock names nobody. pid reuse between the check and `kill` cannot be closed on macOS; the start is read again just before.

## Technical Notes

- The CLI already handles SIGINT/SIGTERM for a clean (signals module, child process groups) and stops between items; `ddc scan` installs no handler (menu bar ADR §6 item 4), so a stopped scan dies at once.
- The app has a single-caller rule for cleans; Stop is not a clean, so it is a separate CLI verb the app calls; it waits in the runner's queue like any other run.
- Start-time primitives exist on macOS (`PROC_PIDTBSDINFO`) and Linux (`/proc/<pid>/stat` field 22).

## Dependencies

- None.

---
type: story
id: "story-scheduled-scans-on-linux-through-a-systemd-user-timer"
epic: "DDC-010"
title: "Scheduled scans on Linux through a systemd user timer"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-02
external_refs: {}
tags: [linux, schedule, cli, capability, systemd]
code_refs: []
specs: []
started_at: null
closed_at: "2026-11-02T16:59:57.304Z"
plan_updated_at: null
---

# Scheduled scans on Linux through a systemd user timer

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | done |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

On macOS, these commands manage a per-user LaunchAgent (`crates/ddc-cli/src/platform/launchd.rs`):
- `ddc schedule install --interval daily|weekly [--threshold SIZE]` writes it, and it runs `scan --background`;
- `ddc schedule uninstall` removes it;
- `ddc schedule status` reports it.

On Linux all three refuse with exit 69, because Linux does not declare `schedule`. This story answers DDC-007's open question on scheduled scans for Linux.

On Linux the same commands manage a systemd user timer.

- **Units.**
  - `deepdrivecleaner-scan.service`: `Type=oneshot`, `ExecStart=<stable path> scan --background [--threshold …]`, `Nice=10`, `IOSchedulingClass=idle`.
  - `deepdrivecleaner-scan.timer`: `OnCalendar=daily` or `weekly`, `Persistent=true` and `RandomizedDelaySec=15min`. `Persistent=true` makes up a run missed while the machine was off. A realtime timer fires on resume after a suspend anyway.
- **Where the units go.** The user manager's search path is fixed by the manager's own environment when it starts. A client's `XDG_CONFIG_HOME` does not move it, so units written under a shell's `XDG_CONFIG_HOME` can be invisible to `daemon-reload`. To avoid this:
  - `ddc` keeps both files in its own data folder (`$HOME/.local/state/deepdrivecleaner/systemd/`, which depends on the home alone);
  - it installs them with `systemctl --user link <abs path>` for the service and `systemctl --user enable --now <abs path>` for the timer, so systemd links them into the manager's own configuration folder;
  - uninstall runs `disable --now` on both (which removes the links), removes the files and reloads.
  
  No root is needed.
- **Status** names:
  - the unit files and whether they are current for this binary;
  - whether the timer is enabled;
  - the next run (`systemctl --user show -p NextElapseUSecRealtime`);
  - the last result (`Result`, `ExecMainStatus`);
  - whether the scan will notify (story "Desktop notifications on Linux through the freedesktop notification service").
- **Where no user systemd runs.** This covers containers, WSL without systemd, OpenRC and runit distros, and some ssh-only servers. There, `schedule install` refuses with exit 69 and the line `ddc: no-user-systemd: …` on stderr (`schedule` has no JSON mode: ADR "Platform seam" §7). Its message names the cron line the user can add instead. It never edits a crontab.
- **Lingering.** Without `loginctl enable-linger`, a user timer runs only while the user has a session. `schedule status` says this and prints the command. `ddc` never runs it.
- **The stable path.** The LaunchAgent refuses a binary inside a cargo `target/` folder (`inside_cargo_target`). On Linux the stable paths are where the install script puts the binary (story "One-line install on macOS and Linux: ddc.sh/install.sh and cargo", DDC-007), `~/.cargo/bin`, and Linuxbrew's prefix.
- **The data folder.** The service runs with the user manager's `HOME` and environment. A user who installed with `--data-dir` or `DDC_DATA_DIR` would otherwise get a timer that writes somewhere else. So the unit pins what the interactive install knew: `ExecStart=… scan --background --data-dir <path>`, and `schedule status` shows that path. The LaunchAgent's `AgentSpec` carries `log_dir` but no data folder today. The plan below decides it for both OSes.
- **Environment.** The service gets the user manager's environment, not the shell's. The decision on which variables go into `Environment=` is made here, early. Story "Cache roots moved by the environment: XDG and each tool's own variable" depends on it. The proposal: `schedule install` writes the honoured cache variables it sees, and `schedule status` lists them.

## Decomposition

- [x] The scheduler seam. One trait in `crates/ddc-cli/src/platform/` with install, uninstall and status, with launchd and systemd behind it, chosen per OS. — done in PR S1 (#101, 0.24.1)
  - Story "Windows scheduled scans through Task Scheduler" (DDC-008) plans the same seam, and whichever lands first builds it.
  - `cmd/schedule.rs` imports `launchd` directly today, and the help text in `cli.rs` says "LaunchAgent". Both become platform-neutral.
- [x] `platform/systemd.rs` in `ddc-cli`: — done in PR S2 (#102, 0.25.0)
  - unit text generation, unit-tested on every OS;
  - `systemctl` behind a trait like `Launchctl`, with a fake for tests;
  - "installed and current" means both files name this binary's stable path and the requested interval, threshold and environment.
- [x] Detection of a user systemd, as plan step S2.5 states: exit 0 or a live state; `offline`, `unknown` or no answer refuse with `no-user-systemd` and print the cron line. — done in S2
- [x] Declare `schedule` on Linux. — done in S2; ADR "Platform seam" §3 and §6
  - Update ADR "Platform seam" §3. No error or JSON schema moves: `schedule` has no JSON mode.
  - The cases runner derives the exit-69 expectation from `CAPABILITIES`, so `ddc schedule status` simply runs on Linux. `schedule install` stays in `not_run`, as on macOS.
- [x] CI on the Linux runner. — the `linux-schedule` job, green on PR and main
  - Enable lingering for the runner user (`sudo loginctl enable-linger $USER`). Export `XDG_RUNTIME_DIR=/run/user/$(id -u)` and the bus address in the step, since the runner sets neither.
  - Copy the test binary out of `target/` to a temporary stable path.
  - Install with `--data-dir <throwaway>`, query, and trigger the service (`systemctl --user start deepdrivecleaner-scan.service`).
  - Check that `last-scan.json` was written in that throwaway data folder, then update and uninstall.
  
  This uses the runner's real, throwaway user manager, with a throwaway `HOME`, and cleans up after itself. Never run it on a developer's machine.
- [x] Owner checks: the timer on a desktop session, a run made up after the machine was off, and a machine without systemd that refuses with the cron line. — in the owner-checks runbook

## Implementation Plan

Planned 2026-11-02 with the planner, from origin/main df71852 (0.24.0). Two PRs, each with its own version bump. The Windows session confirmed it is not building Task Scheduler now, so this story builds the seam and leaves a slot for it. The seam lives in `crates/ddc-cli/src/platform/`, not `cmd/`: the structure test forbids OS-specific crates, modules and `cfg`s outside the platform folders. The DDC-008 story's "in cmd/schedule.rs" should be read that way.

**S1, the seam (patch; no change on macOS):**
1. `platform/scheduler.rs`: `ScheduleSpec` (exe, interval, threshold, log folder, data folder, pinned variables) and `trait Scheduler { install, uninstall, status }`. `launchd::inside_cargo_target`, `scheduled_program`, `same_binary` and `program_state` move here and are shared.
2. `launchd.rs`: `struct Launchd` implements it, with the existing `Launchctl` and its fake.
3. `platform/mod.rs`: `scheduler(env)` picks the backend with a runtime `match PLATFORM`, so both backends compile and are used on every target.
4. `cmd/schedule.rs` goes through the trait. The install, uninstall and status outputs and exits stay unchanged on macOS, and `LABEL` and `plist_path` stay, since the app checks `schedule uninstall`'s exit and reads the plist itself. `cli.rs`'s help and `cases.json`'s reasons stop saying "LaunchAgent".
5. `tests/structure.rs`'s `background_sources_never_reference_cleanup` lists `platform/scheduler.rs` (and `platform/systemd.rs` in S2) beside `platform/launchd.rs`.

**S2, systemd (minor):**
1. `platform/systemd.rs`, compiled everywhere:
   - the unit texts, with `%` written `%%`, and `ExecStart` and `Environment=` values quoted;
   - `trait Systemctl` with a fake;
   - the files kept in `$HOME/.local/state/deepdrivecleaner/systemd/`, a folder that depends on the home alone, so status and uninstall find them whatever the shell's `XDG_STATE_HOME` or `DDC_DATA_DIR` says;
   - `$` written `$$` in `ExecStart`, besides `%` as `%%`.
2. Install first runs the detection (before anything is written, settings included), then `disable`s any earlier timer and service of ours (a link of the same name may point at a moved or deleted folder), writes the files, links the service, reloads, `enable --now` the timer by path, and restarts it.
3. Uninstall runs `disable --now` on both (which removes the links; should a dangling link make `disable` fail on some systemd versions, `link`/`enable --force` is the fallback), removes the files and reloads. With nothing installed it says so.
4. Status with no files makes no `systemctl` call and prints "Not installed": the cases run it. Status reads the last scan from the unit's pinned data folder, not the shell's. A unit that is not current says "run `ddc schedule install` again" (an update through the install script keeps the path but not the unit text). Otherwise it shows:
   - the units and whether they are current for this binary;
   - whether the timer is enabled;
   - the next run;
   - the last result;
   - the data folder;
   - the pinned variables;
   - the linger line;
   - whether the scan notifies.
5. Detection: a manager is there when `systemctl --user is-system-running` exits 0, or prints one of `initializing`, `starting`, `running`, `degraded`, `maintenance`, `stopping`. `offline` (not booted under systemd: WSL without it, containers), `unknown`, no output, a missing binary or a failed connect mean `no-user-systemd`. When `XDG_RUNTIME_DIR` is unset (ssh without `pam_systemd`) the message names that export as well as the cron line.
6. Linux declares `schedule`, and ADR "Platform seam" §3 says so, naming `no-user-systemd` as a run-time unavailability (`EX_UNAVAILABLE`), distinct from the static `capability-absent`. No schema moves.
7. `install.sh --uninstall` refuses on Linux too while our systemd units name its binary (today it checks only the macOS plist), so it never removes the binary from under a live timer. This lands with S2 or before it.
8. A CI job `linux-schedule`, with `timeout-minutes` and `continue-on-error` until it has been green for a week (this story's close flips it, or names who does), runs `scripts/ci-schedule.sh`, which refuses outside CI. The service runs with the runner's own `HOME` (it is never pinned), so the forced run scans the runner's home. The uninstall runs in a trap and in an `if: always()` step, so a failed step leaves no timer. The script:
   - enables linger;
   - sets `XDG_RUNTIME_DIR` and the bus address;
   - copies the binary to a temporary stable path;
   - runs install, status and a forced run with `DDC_DATA_DIR` set to a temporary folder;
   - checks `last-scan.json` and `logs/background.log`;
   - checks `OnCalendar` after a weekly install;
   - uninstalls twice.

**Decisions made here:**
- **The data folder.** It is always pinned on Linux, as `--data-dir <path>` in `ExecStart`, because the manager's `XDG_STATE_HOME` can differ from the shell's. On macOS it goes into the plist only when it is not the default (S2), so default plists stay as they are. `schedule install` takes no new flag; it honours `DDC_DATA_DIR`.
- **`Environment=`.** It lists the variables ddc honours that are set at install: `DOCKER_CONFIG` and `CLAUDE_CODE_TMPDIR` for now. `TMPDIR` is not pinned: a session's value pinned at install may not exist later. Story "Cache roots moved by the environment…" extends the list. Never pinned: `HOME`, `PATH`, `DDC_DATA_DIR`, `XDG_STATE_HOME` (the data folder pin replaces them), `XDG_RUNTIME_DIR` and `DBUS_*`.
- **`no-user-systemd`.** Exit 69 (`EX_UNAVAILABLE`), the `ddc:` line on stderr, and the cron line on stdout. Nothing is written, settings included. The error envelope comes when `schedule` gets a JSON mode.
- **Release notes of S2** name the macOS change: a plist now pins `--data-dir` when the data folder is not the default.
- **"Current".** The unit's program is the same binary as this one, and both files equal what this binary would write for their own parameters.



## Acceptance Criteria

- [x] `ddc schedule install --interval daily` on Linux links both units, enables the timer, and `ddc schedule status` shows its next run (CI test with a real user manager). — evidence: linux-schedule job green on PR #102 (run 41477909935) and on main 225d5f4 (run 41483961461); scripts/ci-schedule.sh (install, `Enabled: yes`, `Next:` a real time); crates/ddc-cli/src/platform/systemd.rs `install_and_uninstall_are_idempotent`
- [x] A forced run of the service writes `last-scan.json` and logs into the data folder that the install named, and `schedule status` shows that folder (CI test). — evidence: linux-schedule job green on PR #102 (run 41477909935) and on main 225d5f4 (run 41483961461); scripts/ci-schedule.sh (`last-scan.json`, `logs/background.log`, `Result: success`, status without `DDC_DATA_DIR` shows the last scan from the pinned folder)
- [x] `ddc schedule uninstall` disables both units (removing their links) and removes the files. Run again, it is a no-op that says so (CI test). — evidence: linux-schedule job green on PR #102 (run 41477909935) and on main 225d5f4 (run 41483961461); scripts/ci-schedule.sh (uninstall twice); systemd.rs `uninstall_cleans_a_link_left_without_files`, `a_failed_disable_relinks_ours_and_tries_again`, `uninstall_without_a_manager_keeps_the_files_and_says_why`
- [x] Without a user systemd, `ddc schedule install` refuses with exit 69, `ddc: no-user-systemd` on stderr and a printed cron line, and writes nothing (test with the fake). — evidence: crates/ddc-cli/src/cmd/schedule.rs `no_user_systemd_refuses_before_anything_is_written` (settings bytes unchanged, only the probe called)
- [x] `CAPABILITIES` for Linux holds `schedule`, and the cases table runs `ddc schedule status` on Linux. — evidence: crates/ddc-core/src/platform/linux/mod.rs (`Capability::Schedule`); crates/ddc-cli/tests/cases.json `schedule status` runs on Linux (PR #102 checks)
- [x] On Linux, with our units naming the installed binary, `install.sh --uninstall` exits 4 and removes nothing (shell test of the install script). — evidence: packaging/install/test_install.py `test_uninstall_stops_on_the_timer_that_runs_it` and the escaped-quote and `%%`/`$$` tests

## Final Summary

**What changed.**
- **PR S1 (#101, 0.24.1).** One `Scheduler` trait in `crates/ddc-cli/src/platform/`, with launchd behind it. macOS outputs and exits are unchanged, and a test pins the status lines.
- **PR S2 (#102, 0.25.0).** A systemd user timer on Linux:
  - the units live in `~/.local/state/deepdrivecleaner/systemd/` and are linked into the user manager;
  - the unit pins the data folder and the honoured cache variables;
  - detection counts only a live manager. Without one, install refuses with exit 69 and `no-user-systemd` before anything is written, and prints a crontab line;
  - status names the units' state, the next run, the last result, the pinned folder and linger;
  - `install.sh --uninstall` stops on Linux while the timer runs its binary;
  - a CI job installs, runs and removes the timer on the runner's own user manager.

**Why.** Linux had no scheduled scans: `ddc schedule` refused there.

**Tests executed.**
- Native Linux runs: 900 at S1, 914 at S2.
- The install-script tests.
- `linux-schedule` on a real user manager, green on the PR and on main.
- Main's full gate green on 225d5f4.

**Risks and follow-ups.**
- `linux-schedule` stays `continue-on-error` until it has been green for a week. The next session that touches gate.yml after 2026-11-09 makes it required, if its runs stayed green.
- The owner checks are in the runbook.

## Technical Notes

- **Deviations:**
  - `available()` takes the spec, so the refusal carries the crontab line.
  - The `XDG_RUNTIME_DIR` hint is always in the refusal, since that variable may not be read outside `ddc-core`'s platform folder.
  - The service writes no log of its own: the scan writes `background.log`.
  - Install uses `link --force` and `enable --force --now`.
  - `TMPDIR` is not pinned.
- **The CI job** waits for `$XDG_RUNTIME_DIR/systemd/private`, not the D-Bus socket, because `dbus-user-session` may be missing on the runner.

- The rule "never run a real `ddc schedule install` from a dev session" binds here too. Tests use the fake `systemctl`, or the CI runner's throwaway user manager.
- `systemd-run --user --on-calendar` would avoid unit files, but transient units do not survive a reboot.

## Dependencies

- Story "One-line install on macOS and Linux: ddc.sh/install.sh and cargo" (DDC-007), for the stable binary path.
- Story "Windows scheduled scans through Task Scheduler" (DDC-008), which shares the scheduler seam.

## Attachments

-

---

*Last updated: 2026-11-02*

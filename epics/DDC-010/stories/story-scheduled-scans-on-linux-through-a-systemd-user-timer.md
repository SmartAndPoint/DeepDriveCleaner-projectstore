---
type: story
id: "story-scheduled-scans-on-linux-through-a-systemd-user-timer"
epic: "DDC-010"
title: "Scheduled scans on Linux through a systemd user timer"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [linux, schedule, cli, capability, systemd]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Scheduled scans on Linux through a systemd user timer

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | planned |
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
  - `ddc` keeps both files in its own data folder (`$XDG_STATE_HOME/deepdrivecleaner/systemd/`);
  - it installs them with `systemctl --user link <abs path>` for the service and `systemctl --user enable --now <abs path>` for the timer, so systemd links them into the manager's own configuration folder;
  - uninstall runs `disable --now`, unlinks both, removes the files and reloads.
  
  No root is needed.
- **Status** names:
  - the unit files and whether they are current for this binary;
  - whether the timer is enabled;
  - the next run (`systemctl --user show -p NextElapseUSecRealtime`);
  - the last result (`Result`, `ExecMainStatus`);
  - whether the scan will notify (story "Desktop notifications on Linux through the freedesktop notification service").
- **Where no user systemd runs.** This covers containers, WSL without systemd, OpenRC and runit distros, and some ssh-only servers. There, `schedule install` refuses with a stable exit code and a JSON reason, `no-user-systemd`. Its message names the cron line the user can add instead. It never edits a crontab.
- **Lingering.** Without `loginctl enable-linger`, a user timer runs only while the user has a session. `schedule status` says this and prints the command. `ddc` never runs it.
- **The stable path.** The LaunchAgent refuses a binary inside a cargo `target/` folder (`inside_cargo_target`). On Linux the stable paths are where the install script puts the binary (story "One-line install on macOS and Linux: ddc.sh/install.sh and cargo", DDC-007), `~/.cargo/bin`, and Linuxbrew's prefix.
- **The data folder.** The service runs with the user manager's `HOME` and environment. A user who installed with `--data-dir` or `DDC_DATA_DIR` would otherwise get a timer that writes somewhere else. So the unit pins what the interactive install knew: `ExecStart=… scan --background --data-dir <path>`, and `schedule status` shows that path. The LaunchAgent's `AgentSpec` carries `log_dir` but no data folder today. Do the same for it here, or record it as an open question for macOS.
- **Environment.** The service gets the user manager's environment, not the shell's. The decision on which variables go into `Environment=` is made here, early. Story "Cache roots moved by the environment: XDG and each tool's own variable" depends on it. The proposal: `schedule install` writes the honoured cache variables it sees, and `schedule status` lists them.

## Decomposition

- [ ] The scheduler seam. One trait in `crates/ddc-cli/src/platform/` with install, uninstall and status, with launchd and systemd behind it, chosen per OS.
  - Story "Windows scheduled scans through Task Scheduler" (DDC-008) plans the same seam, and whichever lands first builds it.
  - `cmd/schedule.rs` imports `launchd` directly today, and the help text in `cli.rs` says "LaunchAgent". Both become platform-neutral.
- [ ] `platform/systemd.rs` in `ddc-cli`:
  - unit text generation, unit-tested on every OS;
  - `systemctl` behind a trait like `Launchctl`, with a fake for tests;
  - "installed and current" means both files name this binary's stable path and the requested interval, threshold and environment.
- [ ] Detection of a user systemd: `systemctl --user is-system-running` answers, or `$XDG_RUNTIME_DIR/systemd/private` exists. Otherwise refuse with `no-user-systemd` and print the cron line.
- [ ] Declare `schedule` on Linux.
  - Update ADR "Platform seam" §3, and the error and JSON schemas for the new reason.
  - The cases runner derives the exit-69 expectation from `CAPABILITIES`, so `ddc schedule status` simply runs on Linux. `schedule install` stays in `not_run`, as on macOS.
- [ ] CI on the Linux runner.
  - Enable lingering for the runner user (`sudo loginctl enable-linger $USER`). Export `XDG_RUNTIME_DIR=/run/user/$(id -u)` and the bus address in the step, since the runner sets neither.
  - Copy the test binary out of `target/` to a temporary stable path.
  - Install with `--data-dir <throwaway>`, query, and trigger the service (`systemctl --user start deepdrivecleaner-scan.service`).
  - Check that `last-scan.json` was written in that throwaway data folder, then update and uninstall.
  
  This uses the runner's real, throwaway user manager, with a throwaway `HOME`, and cleans up after itself. Never run it on a developer's machine.
- [ ] Owner checks: the timer on a desktop session, a run made up after the machine was off, and a machine without systemd that refuses with the cron line.

## Implementation Plan

## Acceptance Criteria

- [ ] `ddc schedule install --interval daily` on Linux links both units, enables the timer, and `ddc schedule status` shows its next run (CI test with a real user manager).
- [ ] A forced run of the service writes `last-scan.json` and logs into the data folder that the install named, and `schedule status` shows that folder (CI test).
- [ ] `ddc schedule uninstall` disables and unlinks both units and removes the files. Run again, it is a no-op that says so (CI test).
- [ ] Without a user systemd, `ddc schedule install` refuses with `no-user-systemd`, a stable exit code and a printed cron line, and writes nothing (test with the fake).
- [ ] `CAPABILITIES` for Linux holds `schedule`, and the cases table runs `ddc schedule status` on Linux.

## Final Summary

## Technical Notes

- The rule "never run a real `ddc schedule install` from a dev session" binds here too. Tests use the fake `systemctl`, or the CI runner's throwaway user manager.
- `systemd-run --user --on-calendar` would avoid unit files, but transient units do not survive a reboot.

## Dependencies

- Story "One-line install on macOS and Linux: ddc.sh/install.sh and cargo" (DDC-007), for the stable binary path.
- Story "Windows scheduled scans through Task Scheduler" (DDC-008), which shares the scheduler seam.

## Attachments

-

---

*Last updated: 2026-11-01*

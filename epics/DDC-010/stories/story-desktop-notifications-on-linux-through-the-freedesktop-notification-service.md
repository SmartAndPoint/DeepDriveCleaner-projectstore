---
type: story
id: "story-desktop-notifications-on-linux-through-the-freedesktop-notification-service"
epic: "DDC-010"
title: "Desktop notifications on Linux through the freedesktop notification service"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [linux, notifications, cli, capability, dbus]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Desktop notifications on Linux through the freedesktop notification service

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

`ddc scan --background` decides whether to notify (`notify::decide`), builds the message with the clean command for the scanned paths (`notify::message`), and logs `notification: …`. On macOS it then shows the message through the CLI's `SystemNotifier` (`osascript`). On Linux the capability `notifications` is absent, so the decision is logged and nobody sees it (`cmd/background.rs`).

On a Linux desktop the standard path is the freedesktop notification service: the D-Bus method `org.freedesktop.Notifications.Notify` on the session bus. GNOME, KDE, XFCE, dunst and mako all implement it. `notify-send` is a client of it, but is not installed everywhere (`libnotify-bin`).

The CLI is a static musl binary, so a D-Bus client that needs `libdbus` will not do. The pure-Rust `zbus` crate speaks the protocol over the Unix socket in `DBUS_SESSION_BUS_ADDRESS`, or `$XDG_RUNTIME_DIR/bus`.

Where no session bus or no notification server answers, the decision is logged as today and the scan exits 0. This covers ssh-only servers, containers and WSL without WSLg. `schedule status` says that the scan will not notify, and why.

## Decomposition

- [ ] `SystemNotifier` for Linux in `crates/ddc-cli/src/platform/`.
  - Call `Notify` through `zbus`, blocking with a short timeout. App name `DeepDriveCleaner`. The summary and body come from `notify::message`. Urgency is normal.
  - Each call has a hint: `desktop-entry` when a `.desktop` file exists, otherwise none.
- [ ] Choose the client and record it in the PR, with numbers.
  - `zbus` in its blocking mode works without `libnotify-bin`, but it brings an async runtime into a synchronous CLI.
  - `notify-send` as a child process mirrors the macOS path (`osascript`), but it is often missing.
  - The proposal: `zbus` if it adds less than 1 MB to the stripped musl binary. Otherwise use `notify-send` where it exists, and log when it does not.
- [ ] Answers.
  - No bus, or `org.freedesktop.DBus.Error.ServiceUnknown`: log `notification not shown: no notification service` and continue.
  - A server that answers: log the notification id.
- [ ] Actions. The server may offer a "Review" action (`actions` capability). The proposal: no action in the first version. The body names the command, as on macOS. Record the reason: a background scan cannot wait for a click.
- [ ] `ddc schedule status` on Linux says whether a notification service answers now (`GetServerInformation`), and names it.
- [ ] Declare `notifications` on Linux, and update ADR "Platform seam" §3.
  - The test `the_notification_is_logged_with_the_clean_of_the_scanned_paths` keeps its Linux entry where it is skipped for another reason, or runs.
  - The background tests that skip through `lacks(notifications)` run.
- [ ] Tests.
  - A fake notification server on a private bus (`dbus-run-session` with a test-owned `org.freedesktop.Notifications`) receives the summary and body. This runs in the Linux CI job, where `dbus` is installed in the step.
  - The native runs' image (`rust:<toolchain>-bookworm`) has no `dbus-daemon`. Add the `dbus` package to `scripts/native-linux.sh`'s setup, or skip in the environment group with the reason.
  - A run with no bus logs the reason and exits 0.

## Implementation Plan

## Acceptance Criteria

- [ ] A background scan on Linux whose decision is to notify calls `org.freedesktop.Notifications.Notify` with the message that names the clean command for the scanned paths (test against a fake server on a private bus).
- [ ] Without a session bus or a notification server, the scan logs why no notification was shown and exits 0 (test).
- [ ] `ddc schedule status` on Linux says whether a notification will be shown (test).
- [ ] `CAPABILITIES` for Linux holds `notifications`. The static musl binary still builds and passes the release smoke test, and the PR states the binary's size before and after.
- [ ] An owner check on a desktop session (GNOME or KDE) shows the notification from the timer's run.

## Final Summary

## Technical Notes

- A systemd user service gets `DBUS_SESSION_BUS_ADDRESS` from the user manager when a graphical session imported it. Without one, a timer's scan finds no server and logs, which is the intended result.
- WSLg runs a notification server only in some setups. Treat it like any other bus.

## Dependencies

- Story "Scheduled scans on Linux through a systemd user timer", for the status line and the scheduled path. The notifier can land first and work with `scan --background` run by hand.

## Attachments

-

---

*Last updated: 2026-11-01*

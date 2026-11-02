---
type: story
id: "story-busy-name-the-scan-or-clean-that-holds-the-lock-and-offer-to-stop-it"
epic: "DDC-002"
title: "Busy: name the scan or clean that holds the lock, and offer to stop it"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-02
updated: 2026-11-02
external_refs: {}
tags: [app, cli, lock, clean]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
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

- **Who holds it: one record per holder.** Scans share the lock (`try_lock_shared`: the person's scan, the background scan, a clean's dry run, `disk free`'s reads, `keep`), so there can be several holders at once; a clean holds it alone. Each holder writes its own record, `<data-dir>/lock.holders/<pid>-<start>.json` (command, mode shared/exclusive, started, for a clean its progress), after it takes the lock, and removes it when it releases the lock **and** explicitly before it re-raises a fatal signal (`signals::reraise` skips `Drop`). A record whose pid is gone, whose start time differs (pid reuse) or whose executable is not a ddc (`ddc`, or the app's `Contents/Helpers/deepdrivecleaner`) is stale and ignored; stale records are normal after a crash or SIGKILL and are swept by the next holder.
- **A read verb.** `ddc lock status [--json]` lists the live holders. The lock itself is never probed to find them: an exclusive probe would make a concurrent scan fail busy for that instant.
- **Busy as a JSON error.** A command that meets a held lock prints the holders (text) and, with `--json`, an error report with code `busy` and the holders: a new `ErrorCode` value, so the error schema's version bumps. The app reads it on exit 75 (today it shows a fixed line).
- **In the app.** When a clean or a scan cannot start, the sheet and the popover list who holds the lock ("A clean started in Terminal 3 min ago (pid 4101), deleting item 4 of 8") with **Stop** and **Wait**. **Wait** is new: the app polls `ddc lock status` every few seconds and starts its own run when no exclusive holder (for a clean, no holder) is left, until the person closes the sheet. **Stop** runs `ddc lock stop --pid P --started T`, which sends SIGINT only if that exact holder still holds the lock, never SIGKILL, same user only; a holder that changed in between is refused, and the sheet shows the new state. If the holder is the app's own run, the app cancels it by its ticket instead.
- **What Stop does to a clean.** A delete or Trash item finishes and the rest are not run. A tool's command (`brew cleanup`, `go clean -modcache`) is interrupted with its process group and reported failed; a simulator runtime deletion continues in CoreSimulator. The sheet says so before Stop.
- **No holder, no button.** When the lock is free, or every record is stale, nothing is shown; a busy result with no live record shows today's line, without Stop: the app cannot stop what it cannot name.
- **ADR.** Signalling a process the app did not start is new against the menu bar ADR §4 (cancellation by ticket); it gets a dated amendment, and the safety ADR a note that Stop is SIGINT through the CLI only. Windows has no process capabilities yet: `lock status` and `lock stop` are unported there.

## Decomposition

- [ ] Core: per-holder records written after taking the lock, removed on release and before `reraise`, stale-record sweep; `store::holders()` with pid, start time and executable checks; every `try_lock` caller (scan, background scan, clean, clean dry run, `disk free`, `keep`)
- [ ] CLI: `ddc lock status [--json]`, `ddc lock stop --pid --started`; busy errors name the holders; `ErrorCode::Busy` and the error schema bump
- [ ] App: busy state from the error report on exit 75; Stop and Wait (polling `lock status`); the wording about tools and runtimes
- [ ] Menu bar ADR amendment (§4) and safety ADR note
- [ ] Tests

## Implementation Plan

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test (core): two shared holders (a scan and a background scan) each have a record and `holders()` lists both; when one releases, the other's record stays; an exclusive holder lists alone
- [ ] Test (core): a record whose pid is gone, whose start time differs, or whose executable is not a ddc is ignored and swept by the next holder; a clean stopped by SIGINT leaves no record (removed before `reraise`)
- [ ] Test (CLI): `ddc lock status --json` lists live holders and validates against its schema; a second `ddc clean` while one runs exits 75 with the holders in text, and with `--json` an error report with code `busy` (error schema version bumped)
- [ ] Test (CLI): `ddc lock stop --pid P --started T` signals SIGINT to that holder only; it refuses a stale record, a holder whose start time differs, another user's process and a non-ddc pid; it never sends SIGKILL; it is refused as unported on Windows
- [ ] Test (DDCKit): the busy state with holders shows them with Stop and Wait; with none it shows today's line and no Stop; Wait starts the run once `lock status` shows no blocking holder; Stop calls `lock stop` with the shown pid and start time
- [ ] A clean stopped this way finishes its current delete or Trash item and reports the rest as not run (existing cancellation tests keep passing)
- [ ] `make check` and `make ddckit-test` pass
- [ ] Owner check (runbook "Owner checks: manual acceptance across stories"): with a terminal `ddc clean --items` running, the Review window's Clean names it, and Stop ends it after the current item

## Final Summary

## Technical Notes

- The CLI already handles SIGINT/SIGTERM for a clean (signals module, child process groups) and stops between items; `ddc scan` installs no handler (menu bar ADR §4), so a stopped scan dies at once.
- The app has a single-caller rule for cleans; Stop is not a clean, so it is a separate CLI verb the app calls; it waits in the runner's queue like any other run.
- Start-time primitives exist on macOS (`PROC_PIDTBSDINFO`) and Linux (`/proc/<pid>/stat` field 22).

## Dependencies

- None.

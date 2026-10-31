---
type: story
id: "story-release-021-the-review-window-in-the-dock-and-skip-what-a-running-process-holds-open"
epic: "DDC-002"
title: "Release 0.2.1: the Review window in the Dock, and skip what a running process holds open"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-20
updated: 2026-10-31
external_refs: {}
tags: [release, app, safety, in-use]
code_refs: []
specs: []
started_at: "2026-10-20T14:18:21.245Z"
closed_at: "2026-10-31T10:30:46.770Z"
plan_updated_at: "2026-10-20T14:18:21.245Z"
---

# Release 0.2.1: the Review window in the Dock, and skip what a running process holds open

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | done |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

Two problems the owner hit on the first day of 0.2.0 (2026-10-20), on the reference machine, in a clean of 214 items:

1. **The Review window gets lost.** The app is menu-bar only (`LSUIElement`), so its windows have no Dock icon and no place in Cmd-Tab. Once another app is in front, the only way back to the Review window is the menu bar icon, then Review in the popover. The owner: «вот это окно по сути сейчас отвязано от бара и в доке тоже нет иконки никакой».
2. **A clean waits ten minutes on a cache in use.** The clean stopped on `uv cache clean` for its whole 10-minute limit (`NATIVE_TIMEOUT`). uv waits for its cache lock (`~/.cache/uv/.lock`), and three long-running `uv run` servers of the owner's held it. The in-use check of the safety ADR (§5) looks at a process's executable, arguments, working directory and `VIRTUAL_ENV`/`CONDA_PREFIX`, not at its open files. So it saw nothing, and the tool's own command blocked. Nothing was lost: a tool command that times out is recorded as failed, and the delete fallback applies only when the tool is missing. But the clean looked hung, and the bar cannot show why (per-item progress is "CLI progress events", not this story).

**What 0.2.1 does (the owner's choice, 2026-10-20: "Dock + skip what is in use"; per-item progress stays in its own story):**
- While any of the app's titled windows is open (Review, Settings, the About panel), the app is a regular app: its icon is in the Dock and in Cmd-Tab, and it has a main menu. When the last of them closes, it goes back to the menu bar only and gives the focus back to the app that had it.
- Open files count as use. An item removed by deletion (`delete`), or by a tool's own command that has a deletion or Trash fallback (`native`), is skipped as `in-use` when a process of the current user has a file open inside it, naming the process, or its app when it lives in an app bundle. A hit is confirmed by reading that process's files again a moment later, so a passing Spotlight worker does not block a cache. Before each tool command the open files are read afresh, since a long clean reaches the tool's item minutes after it started.
- A move to the Trash never counts open files: the Trash keeps an open file working (safety ADR, user-data amendment).
- **Items cleaned only by their runtime's own prune commands (`native-only`: Docker Desktop, Podman) are exempt from the in-use check altogether**, the existing path-based one included. Their daemon runs inside the finding by design, and the prune command is what knows what is in use. 0.2.0 skips Docker's prune through the daemon's working directory whenever Docker runs (critic, 2026-10-20); 0.2.1 fixes that.
- For uv, the tool's command also gets `UV_LOCK_TIMEOUT=10`, so a lock taken after the check fails in seconds with a journaled error. uv honours it from 0.9.16 (2025-12); an older uv ignores it, and the 600 s `NATIVE_TIMEOUT` stays the backstop.
- Release 0.2.1 through the 0.2.0 pipeline. The cask template does not change; the clean report stays at schema version 2 (no new skip reason).

## Decomposition

- [x] App: the activation policy follows the app's titled windows (Review, Settings, About; never the popover or the status item). On open: `.regular` first, then `makeKeyAndOrderFront`, then `activate()`. On the last `willClose`: `.accessory`. After the close box or ⌘W, then yield the activation, so the previous app and its menu bar come back; after the Review window's Close button, whose destination is the popover, stay active as today (a yield followed by an immediate re-activation can be refused under cooperative activation). The Settings window gets the close observer it lacks today
- [x] App: a click on the Dock icon (a reopen event) brings the open windows to the front, in addition to the existing "scan if due"; the reopen cannot be told from the scheduler's `open`, so the effect is additive. `LaunchMode` covers `(.normal, .reopen)` with a window open
- [x] App: a main menu for the regular state: the app menu (About, Settings…, Hide, Quit ⌘Q), Edit (Undo, Redo, Cut, Copy, Paste, Select All, so the search field's shortcuts work), Window (Minimize ⌘M, Close ⌘W). This is the ⌘W follow-up story "Review and clean window" left open
- [x] Core: the open vnode files of the current user's processes (`proc_pidinfo(PROC_PIDLISTFDS)` sized from `pbi_nfiles` with slack; `proc_pidfdinfo(PROC_PIDFDVNODEPATHINFO)` only for `PROX_FDTYPE_VNODE`; an fd closed in between, EBADF or ENOENT, is skipped; an empty path, an unlinked file, is ignored), in the `sys` tree where `unsafe` is allowed; its own pid and its children excluded. The hidden `diag` command prints the read's duration and fd count
- [x] Core: the guard counts open files for `delete` and `native` items; `native-only` items skip the in-use check entirely; Trash moves never count open files. An open-file hit is confirmed by re-reading that one process after about 250 ms. Before each `native` command the open files are read again for that item. The in-use reason names the app bundle when the process lives in one. The dry run (`clean --items --dry-run`, the sheet's check) predicts the same
- [x] Core: `NativeCommand` takes optional environment variables; `cache.uv` sets `UV_LOCK_TIMEOUT=10`
- [x] Safety ADR: a dated amendment to the in-use definition of §5 (open files for `delete` and `native`; never for a Trash move; `native-only` exempt; the re-read before a tool's command; the confirmation re-read), a dated note on the user-data amendment's sentence about open files, and a note that the scan's default selection does not consult processes (only the clean does)
- [x] Menu-bar ADR: a dated amendment to §2 (regular while a titled window is open, accessory after the last one, the main menu)
- [x] README ("The menu bar app": the Dock icon while a window is open; Safety: "or has a file open inside it", `native-only` exempt) and release notes `docs/releases/0.2.1.md` (the two fixes, the Docker prune fix, do not upgrade while the app is cleaning)
- [x] Version 0.2.1 (Cargo, `MARKETING_VERSION`, `CURRENT_PROJECT_VERSION` 3); `make dist`; publish dry run shown to the owner; publish on the owner's approval

## Implementation Plan

No covering spec; the design is the two ADR amendments of 2026-10-20 (safety §5 and its 0.2.1 entry; menu bar §2). The planner's placement (2026-10-20) extends existing modules only: no new crate, module or skip reason.

1. **sys reader** (`crates/ddc-core/src/sys/procs.rs`): `ProcInfo.open_files`; `PROC_PIDFDVNODEPATHINFO`, `ProcFileInfo` and `VnodeFdInfoWithPath` declared locally (not in libc 0.2.191), with a size assertion; `list_fds` sized from `pbi_nfiles` with slack, `vnode_path` returning `None` for a closed fd or an empty path; empty for this process and its children. Paths are compared lexically, never `stat`ed or canonicalized (a protected path could prompt).
2. **Ports** (`exec/mod.rs`, `exec/system.rs`, test fakes): `ProcessTable::open_files(pid)` for the confirming re-read; `CommandRunner::run` takes environment variables.
3. **Guard** (`exec/guard.rs`): `UseCheck { Exempt, Paths, PathsAndOpenFiles }` from the action (`NativeOnly` exempt; `Delete`, `Native` count open files; `Trash`, `TrashEntry` paths only). Pass 1 is the 0.2.0 path check unchanged; pass 2 collects open-file holders, sleeps 250 ms once, re-reads each and skips on the first still holding a file inside. `Guard::new` takes the process port; the reason names the app bundle when there is one. `before_tool(path)` re-reads the table and runs pass 2 before each `native` command (`exec/mod.rs`, `act`).
4. **Rules** (`class.rs`, `rules/catalog.rs`): `NativeCommand.env`; `cache.uv` sets `UV_LOCK_TIMEOUT=10`. The clean report's `CommandResult` is unchanged (schema 2).
5. **CLI** (`cmd/diag.rs`): `process_snapshot: <N> processes, <M> open files, <T> ms`.
6. **Tests**: core `crates/ddc-core/tests/in_use.rs` with `FakeProcs.open_files` and a `ChangingProcs` fake (confirmation, re-read before a tool, Trash, native-only, reason naming, dry-run parity); `clean_safety.rs` (uv env); CLI `clean_items_cli.rs` with a real `/bin/sleep` holder (the core tests run the executor in-process, where a spawned holder is a child and excluded); `diag_cli.rs`; sys unit tests.
7. **DDCKit**: `DockPresence` (pure decision from titled windows: `.regular` on the first, `[.accessory, .yieldActivation]` on the last close, `[.accessory]` when the close returns to the popover); `LaunchMode` `.reopen(windowOpen:)` with `.frontWindows`.
8. **App**: `DockPresenceController` (show: `.regular` → `makeKeyAndOrderFront` → `activate`; `willClose` per window keyed by `ObjectIdentifier`; yield to the previously active app), `MainMenu` (app, Edit, Window), the Review and Settings controllers through it, `AppDelegate` (main menu, reopen, `.frontWindows`).
9. README, `docs/releases/0.2.1.md`, version 0.2.1 (Cargo, Cargo.lock, `MARKETING_VERSION`, `CURRENT_PROJECT_VERSION` 3); `make check`, `make app-test`; reviewer; commit via /sp with the owner's approval; `make dist`, the dry run shown, the publish on the owner's approval.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [x] While any titled window of the app is open (Review, Settings, About), its icon is in the Dock and it is in Cmd-Tab; after the last one closes, neither shows it, and the app that was active before is active again with its menu bar (DDCKit test of the policy decision from the open windows; manual check) — evidence for the policy decision: `DockPresenceTests.firstWindow`, `.lastClose` (`.accessory` and the yield), `.closeToPopover`, `.unknownClose`
- [ ] Manual (owner, moved to the owner-checks runbook at close, 2026-10-31): the manual half of the criterion above: the Dock icon and Cmd-Tab while a titled window is open, and the previous app active again with its menu bar after the last one closes
- [x] A click on the Dock icon brings the open window to the front; Close on the Review window still returns to the popover, and the popover opens from the icon in both states (DDCKit test of `(.normal, .reopen)` with a window open; manual check) — evidence for the tested half: `LaunchModeTests.reopenWithAWindowOpen` (`.scanIfDue` and `.frontWindows`), `DockPresenceTests.closeToPopover`
- [ ] Manual (owner, moved to the owner-checks runbook at close, 2026-10-31): the manual half of the criterion above: the Dock click brings the window forward, Close on the Review window returns to the popover, and the popover opens from the icon with and without a window open
- [ ] Manual (owner, moved to the owner-checks runbook at close, 2026-10-31): in the regular state the main menu works: ⌘Q quits, ⌘W closes the window, ⌘C / ⌘V / ⌘A work in the search field (manual check)
- [x] Rust test (real process: `/bin/sleep` with stdin from a file inside the finding, killed by a drop guard; temp root canonicalized, since the kernel reports `/private/var/…`): `clean --items` skips the deletable finding as `in-use` with that pid and name, and `clean --items --dry-run` predicts the same; once the process exits, the item is cleaned — evidence: `crates/ddc-cli/tests/clean_items_cli.rs::a_file_held_open_by_a_running_process_is_skipped_until_it_closes`; in the core, `crates/ddc-core/tests/in_use.rs::a_file_held_open_inside_a_deleted_finding_skips_it_and_the_dry_run_says_so`
- [x] Rust test: a tool's own command (a fake tool that records whether it ran) is not started for a finding a process holds open, including a process that starts holding it after the clean began (the re-read before the command) — evidence: `crates/ddc-core/tests/in_use.rs::a_tool_command_never_starts_while_a_process_holds_its_cache_open` (held from the start, and taken after the snapshot); `crates/ddc-cli/tests/clean_items_cli.rs::a_tool_never_starts_on_a_cache_a_running_process_holds_open`
- [x] Rust test: a finding bound for the Trash is moved even while a process holds a file open inside it — evidence: `crates/ddc-core/tests/in_use.rs::a_move_to_the_trash_ignores_open_files_and_a_permanent_delete_does_not`
- [x] Rust test: a `native-only` finding whose path a process uses (its working directory or an open file inside it) still runs its prune commands — evidence: `crates/ddc-core/tests/in_use.rs::a_runtime_prunes_while_its_daemon_works_inside_its_finding`
- [x] Rust test: an open-file hit that is gone on the confirming re-read does not skip the item (a fake process table whose second read is empty) — evidence: `crates/ddc-core/tests/in_use.rs::a_file_closed_by_the_confirming_read_does_not_skip`
- [x] Rust test: the uv rule's command carries `UV_LOCK_TIMEOUT=10` — evidence: `crates/ddc-core/tests/in_use.rs::a_tool_command_never_starts_while_a_process_holds_its_cache_open` asserts the environment of `uv cache clean`; the rule sets it in `crates/ddc-core/src/rules/catalog/shared/caches.rs`
- [x] The open-file read of the user's processes costs at most 1 s on the reference machine (measured with the hidden `diag` command, the number recorded here) — evidence: 2026-10-20, release build: 642 processes, about 7,100 open vnode files, 28 to 76 ms (Technical Notes); 2026-10-31, `deepdrivecleaner diag` of the CLI 0.12.1 on the owner's Mac: `process_snapshot: 443 processes, 4827 open files`, 39 ms, then 14 ms and 14 ms
- [x] The safety ADR (§5 and the user-data amendment) and the menu-bar ADR (§2) carry the dated amendments of the decomposition — evidence: ADR "Cleanup safety model: risk classes, plan and confirmation": §5's in-use item and its Amendments entry "2026-10-20, decided by the owner (0.2.1)", and the dated note on the user-data amendment; ADR "Menu bar app: Swift shell over the ddc CLI", §2, "Amended 2026-10-20 for 0.2.1"
- [ ] Manual (owner, moved to the owner-checks runbook at close, 2026-10-31): on the reference machine, with the owner's `uv run` servers running, a clean with `~/.cache/uv` selected skips it as in use by `uv` within seconds and goes on with the rest
- [x] Manual check on the reference machine: with Docker Desktop running, a dry run of a selected Docker item no longer says "In use" — superseded: the owner-checks runbook's check of "VM images: Docker prune runs only while Docker Desktop answers" (DDC-002) runs it against a real Docker Desktop. The automated half: `crates/ddc-core/tests/in_use.rs::a_runtime_prunes_while_its_daemon_works_inside_its_finding` (a fake Docker backend working inside the finding: the dry run predicts `would-run` and the clean runs both prunes)
- [x] 0.2.1 published like 0.2.0: `make dist`, the dry run shown, the publish on the owner's approval, the cask at 0.2.1 — evidence: tag v0.2.1 at 9f0d970 ("release: prepare 0.2.1 with the app in the Dock and open-file checks"), `docs/releases/0.2.1.md`; the tap has since moved on with every release to 0.12.1

## Final Summary

**What changed.** Release 0.2.1 (9f0d970), the two fixes the owner asked for after the first day of 0.2.0.
- The app is a regular app while one of its titled windows is open (Review, Settings, About): in the Dock and in Cmd-Tab, with a main menu (app, Edit, Window). After the last one closes it returns to the menu bar and yields the focus to the app that had it, except when the Review window's Close leads to the popover. A Dock click brings the windows forward (`DockPresence` and `LaunchMode` in DDCKit, `DockPresenceController` and `MainMenu` in the app).
- Open files count as use for `delete` and `native` items: the core reads the open vnode files of the user's processes, confirms a hit by reading that process again about 250 ms later, and reads again right before each tool command. A move to the Trash never counts open files; `native-only` items skip the in-use check altogether, which fixed Docker's prune being skipped whenever Docker ran. `cache.uv` runs with `UV_LOCK_TIMEOUT=10`.
- The hidden `diag` command prints the open-file read's process count, file count and time.
- Dated amendments to the safety ADR's §5 and the menu bar ADR's §2; README and release notes.

**Why.** On 2026-10-20 a clean of 214 items waited the full 10-minute timeout on `uv cache clean`, because running `uv run` servers held uv's cache lock through an open file the in-use check did not see; and the Review window could not be found once another app was in front.

**Tests.** The evidence above; the Rust tests run in the Gate's `gate` job on macOS and the DDCKit tests in its `app` job, both green on main at 2cdae98 (2026-10-31). The open-file read was measured again on 2026-10-31 at 14 to 39 ms. Closed under the owner's rule of 2026-10-30: the manual checks moved to the runbook "Owner checks: manual acceptance across stories".

**Risks and follow-ups.**
- A process that opens a file inside a finding during a tool's command is not seen; uv then fails within `UV_LOCK_TIMEOUT`, any other tool within the 600 s `NATIVE_TIMEOUT`.
- The Dock and Cmd-Tab behaviour, the main menu and the uv skip on the reference machine are manual checks in the runbook.

## Technical Notes

- Switching an `LSUIElement` app to `.regular` at run time is the usual pattern for menu bar apps with windows. The app then needs a main menu; without one, ⌘Q, ⌘W and the Edit shortcuts do nothing, which is also why ⌘C/⌘V in the search field may fail today.
- Open files are read with `proc_pidinfo` and `proc_pidfdinfo` for processes of the same user only; another user's processes are not readable without root, and ddc never runs as root. Unix sockets are not vnode files and are not seen (Docker's `*.sock`), which is fine with `native-only` exempt.
- The snapshot for `delete` items is taken when the clean starts, as the other in-use inputs are. A `native` item reads the open files again right before its command. A process that opens a file in a finding during the command is not seen; uv then fails within `UV_LOCK_TIMEOUT` (uv 0.9.16 and later), any other tool within `NATIVE_TIMEOUT`.
- uv holds its cache lock through an open file in `~/.cache/uv` (fd on `.lock`, process name `uv`; verified on the reference machine), so the open-file check covers it without a uv-specific in-use rule. `uv cache clean --force` is never used: it ignores the lock and would remove environments that running processes use.
- A `native` item with a Trash fallback (`toolchain.pyenv`, `toolchain.rustup`) is skipped on an open file too, although its fallback would be a Trash move: the tool's command is the point of the rule, and it runs while the tool's files are in use.
- Measured by the critic on the reference machine: about 600 processes and 26,000 open fds; `lsof` reads them in 0.5 s, and a direct `proc_pidfdinfo` pass is cheaper. A machine with many browser processes can double it, hence the 1 s budget.
- Measured on the reference machine (2026-10-20, release build, `ddc diag`): 642 processes of the user, about 7,100 open vnode files, read in 28 to 76 ms; the debug build, 41 to 53 ms. Well inside the 1 s budget.
- 0.2.1 inherits none of the open manual checks of the 0.2.0 stories; they continue on the current build, which becomes 0.2.1. Release precondition as for 0.2.0: this story has passed its reviewer.
- Per-item progress in the cleaning sheet is "CLI progress events" (planned), not this story.

## Dependencies

- [[epics/DDC-002/stories/story-distribution-the-020-cask-with-the-app|Distribution: the 0.2.0 cask with the app]] (the release pipeline)
- [[epics/DDC-002/stories/story-review-and-clean-window|Review and clean window]]
- ADR "Cleanup safety model: risk classes, plan and confirmation", §5 and the user-data amendment
- ADR "Menu bar app: Swift shell over the ddc CLI", §2
- Not this story: [[epics/DDC-002/stories/story-cli-progress-events|CLI progress events]] (per-item progress in the cleaning sheet)

## Attachments

-

---

*Last updated: 2026-10-31*

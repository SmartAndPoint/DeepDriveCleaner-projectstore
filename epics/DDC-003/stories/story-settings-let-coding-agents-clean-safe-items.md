---
type: story
id: "story-settings-let-coding-agents-clean-safe-items"
epic: "DDC-003"
title: "Settings: let coding agents clean safe items"
status: in-progress
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-14
updated: 2026-11-15
external_refs: {}
tags: [cli, app, settings, agents, consent]
code_refs: []
specs: []
no_spec: "the contract is this story's decisions D1 to D6 (the retry bound, exit codes and write order included), recorded in the dated notes of 2026-11-15 in ADR 'Agent footprint' §7, the menu bar ADR §1 and §6 item 6, and the safety ADR's journal rule"
# variant: test  (a test story; its guide is story.test)
started_at: "2026-11-15T06:08:06.345Z"
closed_at: null
plan_updated_at: "2026-11-15T06:08:06.345Z"
status_changed_at: "2026-11-15T06:08:06.345Z"
---

# Settings: let coding agents clean safe items

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | in-progress |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The person's switch for ADR "Agent footprint" §7: `ddc agent clean` removes `safe` items only while `settings.json` has `agents_clean_safe: true`, and story "Clean: --agent bounded consent" reads the key but gives no way to switch it except editing the file. This story adds the switch the ADR names.

- `ddc settings set agents_clean_safe on|off`: the one CLI writer of the key, through the settings store's read-modify-write; it records the change in the action journal (a `settings-set` line, journal schema bump) and prints the new state.
- The app's Settings shows "Let coding agents clean safe items", off by default, with the risk in words and a link to the disclaimer; switching it runs the CLI command, never writes the file itself. While it is on, the popover names it in one line, and Settings switches it off in one place.
- The menu bar ADR §6 item 6 ("The CLI writes [settings.json] only to clear a stale scheduler record") gets a dated amendment for this second writer.

## Decomposition

- [ ] PR 0, vault: the plan and the ADRs' dated notes
- [ ] PR 1, core: `settings::edit` with `settings.lock`, `set_agents_clean_safe` with its rollback, `write_atomic` keeping the mode, the `settings-set` journal kind (schema 14)
- [ ] PR 2, CLI: `ddc settings set agents_clean_safe on|off`, the guard on `on`, registries, README
- [ ] PR 3a, DDCKit: `SettingsStore.update` holds `settings.lock`
- [ ] PR 3b, DDCKit and app: the toggle, the popover line, ContractTests

## Implementation Plan

Planner run 2026-11-15 against `origin/main` after PR #153; critic pass the same day (revise, answered here). The owner picked the guard on `on` (2026-11-15, D5).

**Decisions of the plan gate** (the dated ADR notes of 2026-11-15 carry them):

- **D1, no ddc lock.** `ddc settings set` takes neither the shared nor the exclusive lock and never waits for one, for `on` and `off`. `ddc agent clean` reads the setting again under its exclusive lock, so an `off` written before that read is seen; one written after it holds from the next run, the window story "Clean: --agent bounded consent" accepted. Waiting would only delay the person's `off`, and `store::try_lock` has no blocking mode, so an exclusive lock would refuse the toggle during every scan. A running agent clean is stopped with `ddc lock stop` or the Busy panel's Stop. The command never exits 75. Precedent: `notify ack` and `clear_scheduler` take no lock.
- **D2, two writers.** The app never writes `agents_clean_safe` from its own state (`SettingsFile.agentsCleanSafe` is `private(set)` and round-trips as read); this command is the one writer that changes it. A lost `off` would be a consent bug: `SettingsStore.update` could write back a `true` it read before the CLI's `off`. So every writer of `settings.json` holds a sidecar advisory lock, `<data-dir>/settings.lock` (created with the data folder when absent; never removed), around read, edit and atomic rename: Rust's `settings::edit` (`File::try_lock`, retried for about 1 s, then `SettingsError::Busy`: exit 1, never 75), `clear_scheduler` moved onto it (so `ddc schedule install` can now fail with that error, exit 1), and Swift's `SettingsStore.update` (`flock(LOCK_EX|LOCK_NB)`, the same bound, then `problem` and no write). Rust's `File::try_lock` is `flock(2)` on macOS and Linux, so the two exclude each other; a comment at `DataDirectory.settingsLock` says so. Where the file system refuses the lock (`ENOTSUP`, or `ENOLCK` on NFS: a network home), the writer goes on without it and warns: no worse than today. The app holds it only inside `update`, never across a CLI run. After the command the app reloads `settings.json` whatever the exit, so the toggle shows what the CLI left.
- **D3, the write and the journal, in the core.** `settings::set_agents_clean_safe(env, on, journal: &mut dyn SettingsJournal) -> Result<Change, _>`, so the order and the rollback are tested in the core with a journal that fails: (1) take `settings.lock`, creating the data folder if needed; (2) load (unreadable, null or a newer schema: error, nothing written); (3) unchanged: release, `Change::Unchanged`, nothing opened or created, no journal; (4) open the journal (failure: error, the file as it was); (5) write the new settings through `write_atomic`, which now keeps an existing file's mode (Unix); with no file, `on` writes `{"schema_version": 1, "agents_clean_safe": true}`; (6) append one line of kind `settings-set`: `{schema_version, kind, ts, key: "agents_clean_safe", value, previous}`; (7) if the append fails after `on`, the previous settings are written back under the same hold (a file that was absent becomes `{"schema_version": 1}`, which reads as the defaults; nothing is ever removed) and it returns the error, so the opt-in is never on without its record; after `off`, off stays and the error says the line is missing. Journal schema 13 → 14 (the kind `settings-set`); `ddc changes`' lenient reader skips it; no app code reads the journal, so DDCKit's decoders stay. `settings.rs`' module doc ("writes it only to clear a stale scheduler record") changes with it.
- **D4, the command.** `ddc settings set agents_clean_safe on|off`: `SettingKey` is a clap value enum with that one value, so there is no generic writer. Text only, one line `agents_clean_safe: on|off`, plus `(unchanged)`; no `--json` (the app reads the file; `ddc agent status --json` reports the key). Before the lock it reads the file once, only to skip the question when `on` is already on; whether anything is written is decided under the lock (D3), for `off` too. Exit 0 done or unchanged; 1 an error of D2 or D3, or a declined answer (`Nothing changed.`, as `keep remove`); 2 a usage error or an `on` refused without a terminal; 69 through the gate; never 75. Gate: no capability. `--data-dir` hidden, as on `agent clean`.
- **D5, who switches it on (the owner, 2026-11-15).** At a terminal, `on` names the risk and the disclaimer link and asks `[y/N]`, default No; `--approved` there answers the question. Without a terminal it needs `--approved`, which only the app's toggle passes: a yes in a chat is never enough for it, unlike `skill install --approved`. `--approved` is hidden from `--help`; the refusal says `ddc settings set: switching this on needs you at a terminal, or DeepDriveCleaner's Settings` and never names the flag; the README does not name it. `off` never asks. Nothing under `agents/`, no runnable block of the guidance and no agent-facing text names `settings set` (`render::agent_clean` keeps "once the person allows it in DeepDriveCleaner's settings").
- **D6, the app.** A "Coding agents" section in Settings → Automation, outside the schedule's `.disabled`, and disabled with `SettingsProblem` while `store.problem` is set, as the other panes: the toggle "Let coding agents clean safe items", a help line, the risk with "Read the disclaimer" (`DISCLAIMER.md`), and the note that a `ddc` older than this app stops reading `settings.json` while it is on. The toggle with its risk text is the person's consent; there is no second sheet. While the command runs the toggle is disabled; a failure shows the CLI's last stderr line. `CLIRunner.Kind.settingsSet` runs in a lane of its own, so `off` never waits behind a queued scan or clean; the lane serialises a quick on and off (a dated note to the lane rule in the menu bar ADR §1 says why, since it lasts milliseconds). The popover shows "Coding agents may clean safe items" while the key is on; it only opens Settings → Automation through its own closure (`openAgentsOptIn`; `openNextScan` keeps its one caller).

**PR 0, vault**: this plan and the dated notes (menu bar ADR §1 lane rule and §6 item 6, Agent footprint §7, the safety ADR's "A Settings edit writes no journal line").

**PR 1, core (Rust).** `store::write_atomic` keeps an existing file's mode; `settings::{SETTINGS_LOCK, edit, set_agents_clean_safe, Change, SettingsError::Busy}`, `clear_scheduler` on `edit`, the module doc; `exec/journal.rs`: `JournalKind::SettingsSet`, `SettingsSetEntry`, the `SettingsJournal` trait for `FileJournal` and a memory journal that can fail, schema 14 and every pinned version moved; `changes/journal.rs` doc and test; tests in `crates/ddc-core/tests/settings.rs`.

**PR 2, CLI.** `cli.rs` (`Command::Settings`, `SettingsAction::Set`, `SettingKey`, `OnOff`, hidden `--approved` and `--data-dir`), `cmd/settings.rs` (the D4 read, the D5 guard through `prompt::confirm_area_action`, `settings::set_agents_clean_safe` with the file journal, the line), `cmd/mod.rs`, `gate.rs` (no capability, its list); registries as `agent clean` in #153: `cases_cli.rs` (`COMMANDS`, the allowlist with `--data-dir` inside the temporary root, its refusal test), `cases.json`, `unported_cli.rs`; `structure.rs` (no ddc lock, one `set_agents_clean_safe(`; nothing under `agents/` names `settings set`); README (a `ddc settings` section without `--approved`, the Settings table row, the data folder's `settings.lock`, the journal row); tests in `crates/ddc-cli/tests/settings_set_cli.rs`, built on every platform with only the mode check Unix only; while Windows is not supported they skip there with the platform and are listed in `.config/native-skips/`.

**PR 3a, DDCKit: the store's lock.** `SettingsStore.update` holds `settings.lock` (`DataDirectory.settingsLock`, the name pinned to Rust's `SETTINGS_LOCK`); its header loses the "one lost scheduler clear" paragraph.

**PR 3b, DDCKit and app: the toggle and the popover line.** `CLIRunner.Kind.settingsSet` and its lane; `AgentsOptIn.swift`, the one owner of `settings set agents_clean_safe on --approved` and `… off`, reloading the store after the run; `SettingsModel` (runner injected, `agentsCleanSafe`, running and failure, `setAgentsCleanSafe`, never through `store.update`); `SharedSettings.agentsCleanSafe`; `PopoverModel.agentsOptInLine`; `AutomationSettingsView.swift`, `PopoverView.swift`, `StatusItemController.swift`, `AppDelegate.swift`; ContractTests: the toggle is the one caller, nothing scheduled, the popover or AppDelegate names the command, only `SettingsFile.swift` assigns `agentsCleanSafe`, the Swift disclaimer URL and lock name equal Rust's.

No version bump: releases are batched and on hold until the owner asks.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test (CLI): `ddc settings set agents_clean_safe on --approved` writes the key, keeps every other key's value and, on Unix, the file's mode, and journals one `settings-set` line (journal schema 14); with no file it writes one holding only the version and the key; `off` removes the key and journals one line; a call that changes nothing writes no `settings.json` and no journal line (the data folder and `settings.lock` may appear); an unreadable `settings.json` or a journal that cannot be opened exits 1 and leaves the file as it was; it exits 0, never 75, while a clean holds the lock
- [ ] Test (CLI): `on` without a terminal and without `--approved` exits 2, writes nothing, and its message and `--help` never name `--approved`; a declined `[y/N]` exits 1 and writes nothing; `off` never asks; nothing under `agents/` names `settings set`
- [ ] Test (core): `settings::edit` with `settings.lock` held elsewhere gives `Busy` within its bound and writes nothing; `clear_scheduler` keeps every other field and the mode; a journal that fails on append after `on` leaves the settings as they were (an absent file reads as the defaults)
- [ ] Test (DDCKit): switching the toggle runs `settings set agents_clean_safe on --approved` or `… off` once, and the toggle then shows the file as the CLI left it, a failed run included; the app's own Settings writes never change the key and hold `settings.lock` (a held lock writes nothing and sets `problem`); the popover line shows only while `settings.json` holds the key; a contract test pins the Automation toggle as the one caller and that no scheduled path, the popover or AppDelegate names the command
- [ ] The menu bar ADR §1 (the lane rule) and §6 item 6, the Agent footprint ADR §7 and the safety ADR's journal rule carry dated notes for the second writer, `settings.lock`, the lock-free switch, the guard on `on` and the `settings-set` kind
- [ ] Owner check (runbook "Owner checks: manual acceptance across stories"): the toggle's words and the disclaimer link read right; at a terminal, `on` names the risk and asks; with it on, an agent's `ddc agent clean` removes the safe items, and with it off removes nothing

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Status log

- 2026-11-15T06:08:06.345Z status: planned → in-progress — work started (plan gate)

## Technical Notes

- Split out of story "Clean: --agent bounded consent" at its plan gate (2026-11-14, the owner's pick): that story reads `agents_clean_safe`; this one writes it.
- Once the key is on, a `ddc` older than story A's release refuses the whole file; the toggle's text says so.
- ~~Whether `ddc settings set agents_clean_safe off` waits for the exclusive lock (so a running `ddc agent clean` finishes first) is this story's plan gate.~~ *Decided 2026-11-15: no ddc lock (D1).*
- ~~Two writers of `settings.json` then exist (the app's `SettingsStore.update` and this command): both read, change one key and replace the file atomically; the race and its handling are this story's plan gate.~~ *Decided 2026-11-15: every writer holds `settings.lock` (D2).*
- An agent run in a pseudo-terminal sees the `[y/N]` meant for the person, and nothing tells its "y" from a person's; `--approved` stays hidden, so an honest agent is not led to it. This is a property of the owner's choice (D5), not a defect.
- The hidden `--data-dir` (and `DDC_DATA_DIR`) still lets an agent point both commands at a folder of its own; the bound is the owner's rule for an honest agent (ADR §7, note of 2026-11-14), and this story does not close that.

## Dependencies

- [[epics/DDC-003/stories/story-clean-agent-bounded-consent|Clean: --agent bounded consent]]: blocking (the key, `SettingsFile`)

## Attachments

-

---

*Last updated: 2026-11-15*

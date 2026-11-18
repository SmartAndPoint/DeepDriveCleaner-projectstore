---
type: spec
id: "settings-file-writers"
alias: settings-writers
title: "Settings file writers"
status: active
date: 2026-11-15
updated: 2026-11-17
authors: ["Evgenii Konev"]
tags: [settings, agents, consent, cli, app]
external_refs: {}
adr: ["menu-bar-app-swift-shell-over-the-ddc-cli", "agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents", "cleanup-safety-model-risk-classes-plan-and-confirmation"]
stories: ["DDC-003/story-settings-let-coding-agents-clean-safe-items", "DDC-003/story-agent-guidance-skill-and-agents-snippet"]
review_status: reviewed
reviewed_at: "2026-11-15"
---

# Settings file writers

| Field | Value |
|---|---|
| **Status** | active |
| **Date** | 2026-11-15 |
| **Covers** | [[epics/DDC-003/stories/story-settings-let-coding-agents-clean-safe-items\|Settings: let coding agents clean safe items]] |

Spec acceptance is **additive** to the covered stories' own acceptance criteria
— a story is done only when both are green (ADR-007).

---

## How we solve

`<data-dir>/settings.json` has more than one writer: the app's Settings, the CLI's clear of a stale scheduler record, and the switch of `agents_clean_safe`. [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]] §6 item 6 (amendment of 2026-11-15) makes every writer hold one sidecar advisory lock, `settings.lock`, and names `ddc settings set` the one writer that changes the opt-in; its §1 note of the same date gives that command a lane of its own in the app. [[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]] §7, note of 2026-11-15, decides the switch takes no ddc lock and guards `on` (a person at a terminal, or the app's hidden approval). [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]] (keep-remove amendment, note of 2026-11-15) adds the `settings-set` journal line. The decisions D1 to D6 of the covered story's plan are the source of these contracts.

## Behavioral contracts

1. **One writer changes the opt-in.** Only `ddc settings set agents_clean_safe on|off` changes `agents_clean_safe`. Every other write of the file, by the app or the CLI, keeps the key as it read it; the app's toggle switches it only by running that command, and no app code sets the key from its own state: it is only decoded from the file.
   - WHEN the app's Settings edits another key of a file with `agents_clean_safe: true`
   - THEN the key is still true after the write
   - WHEN the file has no key and the app edits another key (edge)
   - THEN the file still has no key

2. **Every writer holds `settings.lock`.** A writer of `settings.json` holds `<data-dir>/settings.lock` while it reads, changes and replaces the file, so no writer loses another's change. One that cannot take it within about a second gives up and changes nothing: `ddc settings set` and `ddc schedule install` (which clears a stale scheduler record) exit 1, never 75; the app says another program is changing the file. Where the file system takes no locks, the writer writes without one: the CLI says so on stderr, the app logs it. The half where the app holds the lock and the CLI writes is checked by the owner (the owner check under Acceptance).
   - WHEN another writer holds the lock for longer than the bound (failure)
   - THEN the switch and the scheduler clear end with "another program is changing the settings", after trying for about a second, and the file is unchanged
   - WHEN the holder lets go within the bound (edge)
   - THEN the waiting write goes through

3. **The switch takes no ddc lock.** `ddc settings set` never takes the lock scans and cleans share, never waits for one and never exits 75. A real `ddc agent clean` reads the opt-in again under its exclusive lock, so an `off` written before that read holds for that run, and one written after it holds from the next.
   - WHEN a clean holds the exclusive ddc lock (edge)
   - THEN `off` exits 0 and the file says off

4. **A switch that changes nothing writes nothing.** When the key already has the asked value, nothing writes `settings.json`, no journal line is written and the journal is not opened; the data folder and `settings.lock` may appear. The output says `(unchanged)`.
   - WHEN `off` runs in a fresh data folder (edge)
   - THEN there is no `settings.json` and no `actions.jsonl` after it
   - WHEN `on` runs while it is on (edge)
   - THEN the bytes of the file are the same and no line is added

5. **A switch keeps the rest of the file.** A switch changes only the one key: every other key keeps its value and the file keeps its permissions. `off` removes the key rather than writing false. With no file, `on` writes a file holding only the schema version and the key.
   - WHEN `on` runs on a private file with roots, rules and a size
   - THEN those values are equal after it and the mode is still private
   - WHEN `on` runs with no file (edge)
   - THEN the file is `{"schema_version": 1, "agents_clean_safe": true}`

6. **Each change is journaled.** A change writes one line of kind `settings-set` to `actions.jsonl` with the key, the new value and the previous one, carrying the journal's `schema_version` (this kind raised it to 14). The journal is opened before the file is written, so a journal that cannot be opened stops the switch with the file as it was: exit 1.
   - WHEN `on` then `off` run
   - THEN two lines follow, `value` true then false, `previous` false then true
   - WHEN the journal's folder cannot be made (failure)
   - THEN exit 1 and the file is unchanged

7. **The opt-in is never on without its record.** When the line fails after `on`, the settings read before are written back under the same hold, and the switch fails; a file that was absent then reads as the defaults, and nothing is removed. When putting it back fails too, the error says the opt-in is still on and to switch it off. When the line fails after `off`, off stays and the error says the journal has no line for it.
   - WHEN the append fails after `on` on an existing file (failure)
   - THEN the file is as it was
   - WHEN the append fails after `on` and the data folder can no longer be written (failure)
   - THEN the error says "still on" and "switch it off"
   - WHEN the append fails after `off` (failure)
   - THEN off stays and the error says the journal has no line for it

8. **A file this ddc cannot read is left alone.** A malformed file, `null`, a newer schema or an unknown key makes every switch fail with exit 1 before anything is written. The error names the file and the reason; for a key this ddc does not know, it says the key may be a newer DeepDriveCleaner's and to update ddc.
   - WHEN the file is `{ not json` (failure)
   - THEN exit 1 and the bytes are the same
   - WHEN the file holds a key this ddc does not know (edge)
   - THEN exit 1, the bytes are the same, and the error names the key and says "update ddc"

9. **`on` needs a person.** When stdin is a terminal, `on` says what it allows, links the disclaimer and asks `[y/N]`, default No; a No exits 1 and writes nothing. Without a terminal it exits 2 and writes nothing unless the app's hidden approval is given; with the approval at a terminal, nothing is asked. When the opt-in is already on, `on` asks nothing, writes nothing and says `(unchanged)`. `off` never asks. As root, every switch exits 2, as the commands that remove or change things do (`clean`, `keep remove`, `agent clean`).
   - WHEN `on` runs on a pseudo-terminal and the answer is `n` (edge)
   - THEN exit 1, "Nothing changed." and no file
   - WHEN `on` runs without a terminal (failure)
   - THEN exit 2 and "needs you at a terminal, or DeepDriveCleaner's Settings"
   - WHEN `off` runs on a pseudo-terminal
   - THEN no question is shown and the exit is 0

10. **Nothing leads an agent to the approval.** The app's approval flag, `--from-app`, is named by no `--help` output, no refusal, no usage error's suggestion and no README text, and shares no name with a flag another command shows: `ddc agent skill install`'s visible `--approved` is refused by `settings set`. The typo filter works per command, so it drops a flag's name from the hints of a command that hides it even while another command shows a flag of that name. No file under `agents/` and no text ddc prints for an agent names `settings set`; the skip reason an agent reads names the person's settings, not the file.
   - WHEN the approval is mistyped as `--from-ap`, `--fromapp` or `--app` (edge)
   - THEN the usage error exits 2 and does not suggest it
   - WHEN `on` is given `--approved` without a terminal (failure)
   - THEN it exits 2 as a usage error and writes nothing
   - WHEN a visible option is mistyped
   - THEN it is still suggested

   > amended 2026-11-17 by DDC-003/story-agent-guidance-skill-and-agents-snippet: MODIFIED — `ddc agent skill install` shows an `--approved` that an agent passes after a yes in the chat, so the app's flag is renamed and the filter works per command. Replaced: "The approval flag is named by no `--help` output, no refusal, no usage error's suggestion and no README text. No file under `agents/` and no text ddc prints for an agent names `settings set`; the skip reason an agent reads names the person's settings, not the file." with the scenario "WHEN the approval is mistyped as `--approve`, `--aproved` or `--approval` (edge)".

11. **The app shows the file as the CLI left it.** After each run of the switch, whatever its exit, the toggle shows what `settings.json` says. A failed run shows the CLI's last stderr line under it, or that the command-line tool could not switch it, or that it is missing. The switch runs beside the queue, so `off` never waits behind a scan. While the file holds the key, the popover shows "Coding agents may clean safe items", which only opens Settings → Automation; it shows nothing when the file is off, absent or unreadable.
   - WHEN the run fails with a stderr line (failure)
   - THEN the toggle shows off and that line
   - WHEN the file goes on, off and unreadable (edge)
   - THEN the popover line shows only while on

## Modules & files on disk

- `crates/ddc-core/src/settings.rs`: `SETTINGS_LOCK`, `edit`, `set_agents_clean_safe`, `clear_scheduler`, `parse` (2, 4 to 8)
- `crates/ddc-core/src/store.rs`: `write_atomic` keeping the mode (5)
- `crates/ddc-core/src/exec/journal.rs`: the `settings-set` kind (6)
- `crates/ddc-core/src/platform/{linux,macos,windows}/errno.rs`: `lock_unsupported` (2)
- `crates/ddc-cli/src/cmd/settings_set.rs`, `crates/ddc-cli/src/cmd/schedule.rs`, `crates/ddc-cli/src/cli.rs` (`without_hidden_hints`): the command, its guard and the scheduler clear (2, 3, 9, 10)
- `crates/ddc-cli/src/prompt.rs`: the `agent-opt-in-off` text (10)
- `app/Packages/DDCKit/Sources/DDCKit/SettingsStore.swift`, `DataDirectory.swift`: the app's hold of the lock (1, 2)
- `app/Packages/DDCKit/Sources/DDCKit/AgentsOptIn.swift`, `SettingsModel.swift`, `CLIRunner.swift`, `PopoverModel.swift`, `SharedSettings.swift`, and the app's `AutomationSettingsView.swift`, `PopoverView.swift`: the toggle and the line (1, 11)

## Testing

- Core, `crates/ddc-core/tests/settings.rs`: 2 (`a_writer_that_holds_the_settings_lock_makes_another_give_up`, `a_writer_that_lets_go_within_the_bound_lets_the_other_write`); 4 (`a_value_already_in_place_opens_and_writes_nothing`); 5 and 6 (`switching_on_and_off_keeps_every_other_key_and_journals_each_change`, `on_without_a_file_writes_one_with_only_the_version_and_the_key`); 7 (`a_journal_line_that_fails_after_on_puts_the_settings_back`, `a_rollback_that_fails_says_the_opt_in_is_still_on`); 8 (`an_unreadable_file_or_journal_leaves_the_settings_as_they_were`, `an_unknown_key_says_to_update_ddc`).
- CLI, `crates/ddc-cli/tests/settings_set_cli.rs`: 3 (`it_takes_no_ddc_lock_so_a_running_clean_never_holds_it_back`); 4 to 6 (`on_and_off_change_the_one_key_and_journal_each_change`, `a_fresh_data_folder_gets_no_settings_file_for_nothing`); 6 and 8 (`an_unreadable_file_or_journal_changes_nothing`); 9 (`at_a_terminal_on_says_what_it_allows_and_asks`, `on_without_a_terminal_is_refused_and_names_no_way_around_it`, `on_when_it_is_on_already_asks_nothing_and_writes_nothing`); 10 (`a_typo_is_never_answered_with_the_hidden_option`); `crates/ddc-cli/tests/structure.rs` `the_settings_switch_takes_no_ddc_lock_and_no_guidance_names_it` (3, 10).
- DDCKit: `SettingsTests` `keepsTheAgentsOptIn`, `neverWritesTheOptIn`, `editsHoldTheSettingsLock` (1, 2); `AgentsOptInTests` `toggleRunsTheCLI`, `failureShowsTheCLI`, `noCLI`, `popoverLine`, `lane` (1, 11); ContractTests `agentsOptInCallers`, `settingsLockName` (1, 2, 11).
- The tests cite no contract by the alias yet; the list above is the map.

## Acceptance

- [x] Each of contracts 1 to 11 has at least one automated test named under Testing that fails when its outcome breaks; the half of 2 where the app holds the lock and the CLI writes is the owner check below — evidence: the Testing list, every named test green in PRs #154 to #157
- [x] The app's lock name and the CLI's are one constant's value, checked by a test (`settingsLockName`); the app's set of errors for a file system without locks is the CLI's macOS set (ENOLCK, ENOTSUP, EOPNOTSUPP), by inspection — evidence: ContractTests `settingsLockName`; `SettingsStore.swift` `takeLock` and `platform/macos/errno.rs` `lock_unsupported`
- [x] A test proves `settings set`'s usage error never suggests the app's approval flag for three likely typos and refuses `--approved`, while `scan --jsn` still suggests `--json`; the filter (`without_hidden_hints`) works per command: it hides a flag's name from the hints of the command that hides it, whatever other commands show (contract 10, amended 2026-11-17) — stories: DDC-003/story-agent-guidance-skill-and-agents-snippet — evidence: `settings_set_cli.rs` `a_typo_is_never_answered_with_the_hidden_option` (PR #172)
- [ ] Owner check (runbook "Owner checks: manual acceptance across stories", section "The switch for coding agents"): with Settings → Automation open and the switch running, a `ddc settings set` from a terminal waits for the app's write or gives up within about a second, and neither change is lost — stories: DDC-003/story-settings-let-coding-agents-clean-safe-items

## References

- [[epics/DDC-003/stories/story-settings-let-coding-agents-clean-safe-items|Settings: let coding agents clean safe items]], its Implementation Plan D1 to D6
- [[epics/DDC-003/stories/story-clean-agent-bounded-consent|Clean: --agent bounded consent]]: the reader of the opt-in

---

*Last updated: 2026-11-15*

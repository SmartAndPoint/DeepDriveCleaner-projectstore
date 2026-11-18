---
type: spec
id: "agent-skill-install"
alias: skill-install
title: "Agent skill install"
status: active
date: 2026-11-17
updated: 2026-11-17
authors: ["Evgenii Konev"]
tags: [agents, skill, agents-md, cli, consent]
external_refs: {}
adr: ["agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents"]
stories: ["DDC-003/story-agent-guidance-skill-and-agents-snippet"]
review_status: reviewed
reviewed_at: "2026-11-17"
---

# Agent skill install

| Field | Value |
|---|---|
| **Status** | active |
| **Date** | 2026-11-17 |
| **Covers** | [[epics/DDC-003/stories/story-agent-guidance-skill-and-agents-snippet\|Agent guidance: skill and AGENTS.md snippet]] |

Spec acceptance is **additive** to the covered stories' own acceptance criteria
— a story is done only when both are green (ADR-007).

---

## How we solve

[[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]] §7, "Installing the guidance", decides `ddc agent skill install` and `ddc agent skill status`. They write the Claude Code skill and the `AGENTS.md` snippet that ship in `agents/`, from copies built into the binary. They write and replace only files they name, delete nothing, and recognise their own skill folder by a marker. At a terminal they ask before writing. Without a terminal they need `--approved`, which an agent passes only after the person said yes in the chat. That flag is not `--yes`: it approves writing `ddc`'s own text, never a removal. The ADR's note of 2026-11-17 records what the plan gate of the covered story settled: the plugin waits for a later story, the marker's hash protects against no one, the app's hidden flag of `ddc settings set` is renamed so the two approvals never share a name, and Windows refuses the command until the port is done. Two neighbouring rules live elsewhere: `.claude/skills` on §3's never-claimed list (the ADR and a rule test), and the typo filter per command ([[specs/settings-file-writers|Settings file writers]], contract 10).

Terms: the **skill folder** is `~/.claude/skills/ddc/`, or `<DIR>/.claude/skills/ddc/` with `--project DIR`. The **shipped files** are the files of `agents/claude-code/ddc/` built into the binary (`SKILL.md` today). The **marker** is `.ddc-skill` in the skill folder. A **marker line** is `<!-- ddc:begin -->` or `<!-- ddc:end -->` alone on its line; the **block** runs from a begin line to the next end line, both included. **This copy** is what this binary would write.

## Behavioral contracts

1. **Where it writes.** `install` writes the skill folder under the home folder, or under `DIR` with `--project DIR`. With `--agents-md FILE` it also writes the block into `FILE`. It writes nowhere else: not the data folder, not `settings.json`, not any other skill. `DIR` must be an existing folder, and `FILE`'s parent folder must exist; otherwise `install` and `status` exit 2, name the path and write nothing. `.claude` and `.claude/skills` are created when missing.
   - WHEN `install --project DIR` runs where DIR does not exist (failure)
   - THEN it exits 2, names DIR, and nothing is written

2. **It writes the shipped files and the marker, and deletes nothing.** In the skill folder it writes each shipped file, then the marker last. Each is replaced atomically and keeps the mode of the file it replaces. No file or folder is ever removed, renamed away or moved to the Trash. A file that is not a shipped file stays as it is, and the output names it: as no longer shipped when the old marker listed it, else as extra. A temporary file an interrupted install left is extra too, left for the person.
   - WHEN the skill folder holds `notes.md` beside an older `SKILL.md` and its marker
   - THEN `SKILL.md` and the marker are new, `notes.md` is unchanged and named as extra
   - WHEN the old marker lists a file this copy no longer ships (edge)
   - THEN that file stays and is named as no longer shipped

3. **It overwrites only its own unedited folder.** `install` writes into a skill folder when the folder is absent, or when the marker parses, every file it lists has the hash the marker gives or equals this copy's file (so an install stopped before its marker counts as unedited), and no shipped file it does not list is there with other bytes. Otherwise the folder is not ours (no marker, or a marker that does not parse) or edited (a listed file is missing or differs from both, or an unlisted shipped file is someone's own). Then `install` writes nothing anywhere, exits 2, and names the folder, the reason and `--replace`. With `--replace` it writes the shipped files and the marker over it, and still deletes nothing; on a folder of its own, `--replace` changes nothing.
   - WHEN `SKILL.md` was edited by hand after an install (edge)
   - THEN `install` exits 2 with "edited", and `install --replace` writes it
   - WHEN the folder was copied by hand and has no marker (edge)
   - THEN `install` exits 2 with "not installed by ddc"

   > amended 2026-11-17 by DDC-003/story-agent-guidance-skill-and-agents-snippet: MODIFIED — the reviewer found a file a later ddc ships, already there as the person's own, would be overwritten. Replaced: "or when the marker parses and every file it lists has the hash the marker gives or equals this copy's file (so an install stopped before its marker counts as unedited). Otherwise the folder is not ours (no marker, or a marker that does not parse) or edited (a listed file is missing or differs from both)."

4. **Symbolic links are refused.** The skill folder, a shipped file or the marker inside it, or `FILE`, when it is a symbolic link, is refused: exit 2, nothing written, even with `--replace`. A link higher up (`~/.claude` itself, say) is followed like any other folder.
   - WHEN `FILE` is a symbolic link to another file (failure)
   - THEN it exits 2 and both files are unchanged

5. **The marker and the versions.** `.ddc-skill` is JSON: the version of the `ddc` that wrote it, and for each file it wrote its name and a 64-bit FNV-1a hash of its bytes in hexadecimal. The hash finds edits; it protects against no one. Line endings are LF in the repository, so the hash is the same on every platform. `SKILL.md`'s frontmatter has `metadata.version`, the version of the binary that ships it; `make bump` writes it with the other version files, and a test checks it equals the crate's version.

6. **The block in `AGENTS.md`: what is refused.** `install --agents-md FILE` refuses, with exit 2 and nothing written anywhere, a file with a begin line and no end line after it, an end line with no begin line before it, more than one begin line or more than one end line, or a CRLF line ending anywhere in it (until the Windows port decides those). A marker line is matched whole: a line that holds the marker among other text is ordinary text.
   - WHEN `FILE` has two begin lines (failure)
   - THEN it exits 2 and `FILE` is unchanged
   - WHEN `FILE` has an end line and no begin line (failure)
   - THEN it exits 2 and `FILE` is unchanged

7. **The block in `AGENTS.md`: what is written.** With a block, `install` replaces it with this copy's block and keeps every byte before and after it; when the old end line was the file's last line without a newline, the new block ends with one, the only byte added. With no block, it appends: a newline when the file does not end in one, then an empty line when its last line is not empty, then the block. An absent or empty `FILE` becomes the block alone. The file is replaced atomically and keeps its mode.
   - WHEN `FILE` has text before and after an older block
   - THEN that text is byte for byte the same and the block is new
   - WHEN `FILE` ends in `text` without a newline and has no block (edge)
   - THEN it becomes `text`, a newline, an empty line and the block
   - WHEN `FILE` already ends in an empty line (edge)
   - THEN no second empty line is added before the block

8. **Consent before any write.** `install` first works out what it would change and checks every refusal of contracts 1, 3, 4 and 6. When something would change, it lists each file and asks `[y/N]` at a terminal, default No; a No exits 1 and writes nothing. With `--approved` it asks nothing, at a terminal or not. Without a terminal and without `--approved` it exits 2, naming `--approved` and saying it is for after the person agreed. When nothing would change, it asks nothing, writes nothing and says the guidance is current. `--approved` shows in `install --help`. As root, `install` exits 2: it never writes files a person then cannot change.
   - WHEN `install` runs with no terminal and no `--approved` (failure)
   - THEN it exits 2 and nothing is written
   - WHEN `install` runs on a pseudo-terminal and the answer is `n` (edge)
   - THEN it exits 1 and nothing is written
   - WHEN this copy is installed already (edge)
   - THEN it exits 0, asks nothing, and no file's modification time changes

9. **`status`: the skill folder.** `ddc agent skill status [--project DIR] [--agents-md FILE] [--json]` writes nothing and asks nothing. For the home skill folder, and the project's with `--project`, it reports one state: `absent`; `current` (unedited, and every file equals this copy's); `older` (unedited, written by another version or from another copy); `edited`; `not-ours`; or `link`. With `not-ours` it reports the `metadata.version` of a `SKILL.md` there, when it has one, so a copy installed by hand shows its version. It lists the extra and no-longer-shipped files.
   - WHEN the home folder has no skill (edge)
   - THEN `status --json` says `absent` and exits 0
   - WHEN a hand copy of an older release is there (edge)
   - THEN it says `not-ours` with that copy's version

10. **`status`: the `AGENTS.md` file.** For `FILE` with `--agents-md` it reports `absent` (no file, or no block), `current` (the block equals this copy's), `differs`, `malformed` (what contract 6 refuses) or `link`. `--json` for both follows `docs/skill.schema.json`. `status` exits 0 for every state, 2 for a path contract 1 rejects, and 1 only when a file cannot be read.
    - WHEN `FILE` holds an older block (edge)
    - THEN it says `differs` and exits 0

11. **No lock, no journal.** Neither command takes the lock scans and cleans share, `settings.lock`, or any other lock, and neither writes the action journal. A running scan or clean never holds them back.

12. **Windows waits.** On Windows both commands are refused like every command except `rules`, until the port is done.

## Modules & files on disk

- `agents/claude-code/ddc/SKILL.md` (`metadata.version`), `agents/AGENTS.snippet.md`: the shipped text (2, 5, 7)
- `crates/ddc-core/src/guidance/`: the embedded copies, the marker and the hash, the folder's and the file's states, the block's parse and replace (2 to 7, 9, 10). Outside `ddc-core/src/agents/`, the read-only harness reader
- `crates/ddc-core/src/store.rs`: `write_atomic`, keeping the mode (2, 7)
- `crates/ddc-cli/src/cli.rs` (`AgentAction::Skill`), `crates/ddc-cli/src/cmd/agent_skill.rs`, `crates/ddc-cli/src/gate.rs`: the commands, consent and refusals (1, 8, 11, 12)
- `docs/skill.schema.json`: `status --json` (9, 10)
- `scripts/bump-version.sh`: `metadata.version` (5)

## Testing

- Core, `crates/ddc-core/tests/guidance.rs`: 2 to 7, 9 and 10 on temporary folders (`an_absent_folder_gets_the_files_then_the_marker`, `an_older_copy_is_replaced_and_other_files_stay`, `an_edited_or_hand_copied_folder_is_refused_unless_replaced`, `a_broken_marker_a_missing_file_or_an_unlisted_one_is_refused`, `links_are_refused_even_with_replace`, `a_malformed_agents_md_is_refused`, `the_block_is_replaced_and_everything_else_kept`, `a_block_is_appended_after_one_empty_line`, `an_agents_md_reports_its_state`, `the_status_report_names_each_state` and more), and the embedded copies equal the files in `agents/` (`the_built_in_copies_are_the_files_in_agents`).
- CLI, `crates/ddc-cli/tests/agent_skill_cli.rs`: 1 (`a_project_gets_its_skill_and_its_agents_md_block`), 3 (`an_edited_skill_is_refused_until_replace`), 4 (`a_linked_agents_md_is_refused_and_nothing_is_written`), 8 (`without_a_terminal_it_needs_approved_and_then_writes_once`, `at_a_terminal_it_lists_the_files_and_asks`), 9 and 10 (`status_reports_each_place_and_writes_nothing`).
- `crates/ddc-cli/tests/structure.rs`: 2 and 11 (`the_guidance_installer_deletes_locks_and_runs_nothing`: no removal, rename, lock, journal or command in the installer), 5 (`the_skill_allows_only_what_reads_and_carries_the_version`); `src/gate.rs` `an_unported_platform_runs_only_rules` (12).
- Each test names the contract it covers in a comment beside it.

## Acceptance

- [x] Each of contracts 1 to 12 has at least one automated test named under Testing that fails when its outcome breaks — stories: DDC-003/story-agent-guidance-skill-and-agents-snippet — evidence: the Testing list, every named test green in PRs #173 and #175
- [x] A test checks that the copies built into the binary are byte for byte the files in `agents/`, so a change to the guidance cannot ship without the binary carrying it — evidence: `guidance.rs` `the_built_in_copies_are_the_files_in_agents` (PR #173)
- [x] `install` and `status` run on Linux and macOS in the Gate; on Windows they are refused and their tests skip, as listed in `.config/native-skips/windows.txt` — evidence: PR #175's Gate; the native Windows leg, 1183 tests, the skips as listed
- [ ] Owner check (runbook "Owner checks: manual acceptance across stories", section "Agent guidance and ddc agent skill install"): `ddc agent skill install` at a terminal lists the files and asks; after a yes, a new Claude Code session finds the skill — stories: DDC-003/story-agent-guidance-skill-and-agents-snippet

## References

- [[epics/DDC-003/stories/story-agent-guidance-skill-and-agents-snippet|Agent guidance: skill and AGENTS.md snippet]], its Implementation Plan
- [[specs/settings-file-writers|Settings file writers]]: the other command with an approval, and the typo filter (its contract 10, amended 2026-11-17)
- [[epics/DDC-003/stories/story-agent-skill-v0-ask-ddc-before-guessing-about-the-disk|Agent skill v0: ask ddc before guessing about the disk]]: the guidance this command installs

---

*Last updated: 2026-11-17*

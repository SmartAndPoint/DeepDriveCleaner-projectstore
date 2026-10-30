---
type: story
id: "story-harness-data-rules"
epic: "DDC-003"
title: "Harness data rules"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-31
external_refs: {}
tags: [rules, agents, safety]
code_refs: ["crates/ddc-core/src/agents/", "crates/ddc-core/src/rules/entries.rs", "crates/ddc-core/src/rules/catalog/macos.rs", "crates/ddc-core/src/rules/listing.rs", "crates/ddc-core/src/rules/visitor.rs", "crates/ddc-core/src/rules/classify.rs", "crates/ddc-core/src/report/contents.rs", "crates/ddc-core/src/report/recheck.rs", "crates/ddc-core/src/report/after_clean.rs", "crates/ddc-core/src/exec/guard.rs", "crates/ddc-core/src/exec/preflight.rs", "crates/ddc-core/src/exec/mod.rs", "crates/ddc-core/src/exec/journal.rs", "crates/ddc-core/src/exec/report.rs", "crates/ddc-core/src/plan.rs", "crates/ddc-core/src/model.rs", "crates/ddc-core/src/progress.rs", "crates/ddc-cli/src/cmd/clean_items.rs", "crates/ddc-cli/src/prompt.rs", "bench/make_baseline.py", "docs/scan.schema.json", "docs/clean.schema.json", "docs/progress.schema.json", "docs/rules.schema.json", "app/Packages/DDCKit/Sources/DDCKit/CleanReport.swift", "app/Packages/DDCKit/Sources/DDCKit/CleanWording.swift", "app/Packages/DDCKit/Sources/DDCKit/SkipSummary.swift", "app/Packages/DDCKit/Sources/DDCKit/ScanReport.swift", "app/Packages/DDCKit/Sources/DDCKit/ProgressEvent.swift", "app/Packages/DDCKit/Sources/DDCKit/TypesRows.swift", "crates/ddc-core/tests/agent_rules.rs", "crates/ddc-core/tests/liveness.rs", "crates/ddc-core/tests/session_entries.rs", "crates/ddc-core/tests/harness_clean.rs", "crates/ddc-cli/tests/rules_cli.rs"]
specs: []
started_at: "2026-10-30T15:46:57.178Z"
closed_at: "2026-10-30T21:47:35.000Z"
plan_updated_at: "2026-10-30T15:46:57.178Z"
---

# Harness data rules

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | done |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

What Claude Code, Codex, opencode and Cursor keep under the home folder gets rules: §3 of the ADR "Agent footprint: worktrees, harness data and bounded consent for agents", as revised on 2026-10-30 for the DDC-002 story "Review window: coding agents' data as a group". That story shows these rules as a "Coding agents" group in the Review window and waits for this one.

- Claude Code's debug logs and legacy folders are `safe`, removed with §5's in-use check.
- Transcripts and file history are **entries**: one finding per project folder (and one for `file-history`), method `none` for the folder, whose sessions are entries that go to the Trash one by one, after the liveness check. A session's transcript, its folder and its set-aside files move together.
- `paste-cache`, `usage-data`, the Codex, opencode and Cursor agent data are `review`, to the Trash. opencode is claimed by subfolder (`storage`, `snapshot`, `log`), never whole: its folder holds `auth.json`, the provider credentials.
- Never claimed: auto memory (`projects/<project>/memory/`), `sessions/`, `shell-snapshots`, the newer per-session folders until classified, Cursor's extensions, plugins, worktrees, plans and chats, Codex's worktrees, and opencode's `auth.json` and `opencode.db`.
- **Liveness** at plan time (the `--items` pruned walk and the dry run) and before each move: per session for Claude Code (its record, a change in the last 10 minutes, an open file; any unreadable record refuses all), per harness for Codex (a held thread lock or a running Codex process), opencode and Cursor, none for `paste-cache` and `usage-data`. A session entry is refused `entry-locked` with `live-session` or `liveness-unknown`, which the scan also sets; a whole-folder item is refused `agent-live`.
- Each rule gets `origin` with `kind: session-data` or `runtime-cache`, and `origin.harness` gains `cursor`.

The paths come from the harnesses' documentation and the research note "Coding-agent harnesses on macOS: where their data lives and how to tell a live session"; the plan gate confirms each on the reference machine by name and size only, and a path that does not exist there enters only with a fixture.

## Decomposition

Rough until the plan gate.

- [x] Known-path rules in `rules/catalog/macos.rs`, with `origin` and the hints of the ADR's table: `agent.claude-code-scratch` (`debug`, `todos`, `statsig`, `logs`; no `shell-snapshots`), `agent.claude-code-file-history` and `agent.claude-code-transcripts` (entries), `agent.claude-code-history`, `agent.codex-sessions`, `agent.codex-runtimes`, `agent.opencode-data` (by subfolder), `agent.cursor-agent-data`. No owner match by process name: §5's in-use check stays, and an owner match would block every path whenever Claude Code runs.
- [x] The never-claimed list in the visitor.
- [x] The entry model, from `ENTRY_RULES` (user data) to a per-rule entry policy: which children are entries (session ids), their companions (re-derived by name under the parent descriptor, moved transcript-last, one journal line per moved path), no size floor, Trash only; `EntryLock` `live-session` and `liveness-unknown`, replacing `recent` for these rules; the scan-report schema bump.
- [x] The liveness guard in `exec/guard.rs`, reusing `agents::claude::sessions` (which learns to report each parse failure), `agents::codex` and `agents::running` (made `pub(crate)`), at plan time and before each move; `SkipReason::AgentLive { harness, reason }` for whole-folder items and the clean-report bump; `running::classify` learns `cursor-agent`.
- [x] A "Coding agents" group in `GROUPS` (`rules/listing.rs`); `ddc rules --json` lists it.
- [x] Plan gate: on the reference machine, names, sizes and times only (`ls`, `du -sk`, `stat`), never contents: which paths exist; the newer per-session folders; the Codex session layout; what `.cursor/projects` holds; with a throwaway session, whether `--resume` writes its record before opening the transcript and whether the record's `sessionId` follows `/clear`.
- [x] `bench/make_baseline.py`: expected rule and class for the new paths.

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

From the planner's pass over origin/main (2026-10-30) and the plan gate on the reference machine (names, sizes and times only), whose results ADR §3 now records under "Settled at the plan gate". The catalog has moved to `rules/catalog/`; the eight rules are macOS rows in `rules/catalog/macos.rs`, not shared ones. Four PRs, each with `make gate` green.

1. **Process facts for liveness** (no schema change). `ProcessTable` gains `process_start` and `open_vnodes`, `SystemProcesses` forwards them to `platform::procs` (the Linux and Windows stubs return nothing). `agents::claude` splits a strict `records` (each unparsable record reported, uid and starts injected) from `sessions`, which `disk agents` keeps unchanged; `codex::threads` and `held` become `pub(crate)`; `running::classify` learns `cursor-agent`. A new pure `agents/liveness.rs` decides a session's and a harness's state from injected facts. Tests: the liveness tables (fake pids and starts, never the test's own pid); `tests/agents.rs` unchanged.
2. **The coding agents rules** (no version bump). `Origin {harness, kind}` in `model.rs`, `Rule.origin` and `listed_only` with the builder; the eight rows in match order, plus unlisted guard rows (the `macos.app-bundle` pattern, sealed) for `.claude/sessions`, `.claude/shell-snapshots`, `.claude/plugins` and `.cursor/{extensions,plugins,plans,chats}`; no row for `.cursor/worktrees` or `.codex/worktrees` (the rules work inside them) nor for `memory/` (inside the project claim, protected by the entry policy); the "Coding agents" group, and `switchable` widened to an entry rule whose class is not `user-data`; `bench/make_baseline.py`; `docs/rules.schema.json`, `bench/golden/rules.json` and the DDCKit rules fixture. The Codex, opencode and Cursor rules ship as method `none` (report only) until PR 4 brings their liveness; scratch (`safe`, delete) and history (`review`, Trash, `listed_only`) work at once under §5's in-use check. ACs 1, 6 and 8, and AC 2 apart from entries.
3. **Session entries in the scan, and the schema bumps.** `rules/entries.rs`: an `EntryPolicy` per rule (`ENTRY_RULES` keeps meaning user data), the `Transcript` and `SessionDir` shapes, companions by name, the `live-session` and `liveness-unknown` locks ahead of `recent` for these rules only. Each `ENTRY_RULES` call site decides whether it means user data or any entry rule (visitor, classify, contents, after-clean, recheck). `report/contents.rs`: entries keyed by the transcript, bytes summed with the companions, no floor for these rules, ties broken by name so the goldens hold; the scan reads liveness lazily, only when a Claude Code entry folder was found. `SkipReason::AgentLive` is declared. `plan.rs`, `exec/preflight.rs` and `exec/mod.rs` stay user-data-only until PR 4, so the dry run and the clean give the same answer for a session entry (`not-an-entry`) in between. Scan 2 → 3, clean 4 → 5, progress 2 → 3, journal 4 → 5 for `agent-live` and the two lock values in `item` lines, declaring the optional `entry` that PR 4's companion lines carry (`ddc keep remove`, DDC-002 "Protected by you: say what it is, and Remove Protection", takes 3 → 4 and is assumed to land first; the five literal pins move: `tests/disk_free.rs`, `tests/clean_safety.rs`, `tests/keep.rs` in ddc-core, `tests/disk_free_cli.rs`, `tests/keep_cli.rs` in ddc-cli); schemas, goldens, `prompt.rs` wording, and DDCKit's decoders, fixtures and contract counts in the same PR. AC 2's entry parts, AC 4 on the scan side, AC 7.
4. **Clean session entries and refuse live harnesses.** `plan.rs`: entry dispatch for policy rules, `action-not-allowed` for a permanent or delete action, `agent-live` from one harness snapshot. `exec/guard.rs`: companions re-derived through `stats::read_dir` and `stat_child` under the same parent descriptor, liveness read again before each move, Codex locks re-read per item. `exec/preflight.rs` the same, so the dry run and the run agree. `exec/mod.rs`: `action_allowed` for policy rules, companions moved first and the transcript last, one journal line per moved path, stop at the first failure. The Codex, opencode and Cursor rules switch to the Trash here. Undo and history readers are checked for one-line-per-item assumptions (none exist on 2026-10-30; a future reader dispatches on `kind` first, as `keep-remove` lines carry `result` as a string and `area-action` lines as an object). ACs 3, 4, 5 and 9.

The user-data suites (`tests/user_data.rs`, `tests/clean_items.rs`, `ddc-loadgen/tests/properties.rs`) run unchanged as the regression net. New tests use the `common::unsupported` guard so the Linux and Windows jobs pass.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [x] Test (fixtures under a fake home): each path of the eight rules is found with its id, class, method and `origin`; `.local/share/opencode/auth.json` and `opencode.db` are in no finding; `every_rule_is_covered_by_a_fixture` stays green; `catalog_invariants`: no `agent.*` rule of class `safe` names `paste-cache`, `usage-data`, `file-history`, `projects` or `shell-snapshots`. — evidence: `crates/ddc-core/tests/agent_rules.rs::each_path_of_the_eight_rules_is_found_with_its_rule_and_origin`; `tests/rules.rs::every_rule_is_covered_by_a_fixture`, `::catalog_invariants`, `::no_safe_agent_rule_names_a_harness_history`
- [x] Test: no finding has a path at or under `projects/<p>/memory/`, `memory/` is never an entry, and `plan_listed` refuses it; `sessions/`, `shell-snapshots/`, `.claude/plugins`, `.cursor/extensions|plugins|worktrees|plans|chats` and `.codex/worktrees` are never findings, while a `node_modules` inside `.codex/worktrees/<w>/` still is; a lone `<s>/` or `<s>.desktop-released.json` without `<s>.jsonl` is no entry; `sessions-index.json` is no entry; `ide.cursor-cache` still claims the desktop app's caches; a `.venv` inside `projects/<p>/<session>/tool-results/` is not a `python.venv` finding. — evidence: `agent_rules.rs::sealed_folders_and_auto_memory_are_never_findings` (with the dry run's reasons); `session_entries.rs::sessions_are_entries_keyed_by_the_transcript_with_their_companions`
- [x] Test: a session entry (`<s>.jsonl` with `<s>/`, `<s>.superseded-1` and `<s>.desktop-released.json`) goes to the Trash as one action, companions first, its `bytes` the sum, one journal line per moved path with `entry` on the companions'; a companion that fails to move stops the item `failed` with `<s>.jsonl` still in place and the moved companions journaled; `--permanent` or `action: delete` on it is `action-not-allowed`; entries are listed with no size floor. — evidence: `crates/ddc-core/tests/harness_clean.rs::a_session_entry_goes_to_the_trash_companions_first_transcript_last`, `::a_companion_that_fails_stops_the_entry_and_the_transcript_stays`, `::a_session_entry_is_never_deleted`; no floor in `session_entries.rs::sessions_are_entries_keyed_by_the_transcript_with_their_companions`
- [x] Test (Claude Code liveness, injected process starts and open files): a fake `sessions/<pid>.json` for a live process gives that session's entry `contents.entries[].lock` `live-session` in the scan, and refuses it `entry-locked` at plan time and in the guard, while another session's entry passes; a record that appears after the plan refuses the entry in the guard; a record-less session changed under 10 minutes ago is `live-session` with `by: recent`; one unparsable record beside parsable live and stale ones, a live record with no `sessionId`, and a missing `sessions/` folder each refuse every Claude Code entry `liveness-unknown`; a record whose process is gone does not block; `paste-cache` is moved while a live session record exists. — evidence: `session_entries.rs::a_live_record_or_an_open_file_locks_its_session_alone`, `::a_session_changed_in_the_last_ten_minutes_is_live_by_recent`, `::records_that_cannot_be_decided_lock_every_session_unknown`; `harness_clean.rs::a_record_written_after_the_plan_keeps_the_session` (one session; another session passing is asserted at the scan and the plan, not at the guard, see the risks), `::a_session_whose_liveness_is_unknown_stays`, `::paste_cache_moves_while_a_session_is_live`; `tests/liveness.rs::records_starts_and_open_files_decide_each_session`
- [x] Test (harness liveness): a fake Codex lock whose `(dev, ino)` an injected open-files list returns, and separately an injected Codex app-server process, refuse `agent.codex-sessions` and `agent.codex-runtimes` with `agent-live`; an injected `opencode` process refuses `agent.opencode-data`; an injected `cursor-agent` process refuses `agent.cursor-agent-data`; with none running each moves. — evidence: `harness_clean.rs::a_live_harness_keeps_its_folders` (plan, dry run and guard; each moves with none running)
- [x] Test: `agent.claude-code-history` is not in a `--yes` plan and `paste-cache` still exists after it; `--yes --class review` is refused `review-with-yes` as today; a plan built without a list never holds a `listed_only` rule's item. — evidence: `agent_rules.rs::history_is_never_in_a_yes_plan_and_outlives_the_clean`, `::yes_with_the_review_class_is_refused`, `::a_plan_without_a_list_never_holds_a_listed_only_item`
- [x] Test: `ddc rules --json` lists the group "Coding agents", switchable, while `user.downloads` stays `switchable: false`; the goldens, the schemas and the DDCKit fixtures and decoders are updated for the scan, clean and progress bumps, and the journal's five test pins move to 5. — evidence: `crates/ddc-cli/tests/rules_cli.rs::rules_json_lists_the_coding_agents_group_with_a_switch`; journal pins at 5 in `tests/keep.rs`, `tests/disk_free.rs`, `tests/clean_safety.rs`, `ddc-cli/tests/disk_free_cli.rs`, `ddc-cli/tests/keep_cli.rs`; DDCKit `ContractTests`, 423 tests
- [x] `bench/make_baseline.py` expects the new rows (rule and class), and its fixture test passes. The run on the reference machine (`ddc scan ~` lists each confirmed path; `python3 -I bench/check_baseline.py`) goes to the owner's manual checks runbook at close (moved there, 2026-10-31). — evidence: `python3 -I -m unittest discover -s bench -p test_make_baseline.py` (automated part); the reference-machine run is in the owner-checks runbook
- [x] `make gate` passes. — evidence: main's Gate run 41131722820 on 58a871a, the story's last commit, ran the whole `make gate` and passed; the branch runs of PRs #12, #18, #21 and #22 passed (the unit tier since PR #20; the native Linux and Windows checks required since PR #19)

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

**What changed.** Eight rules in one "Coding agents" group cover what Claude Code, Codex, opencode and Cursor keep in the home folder, and a session is cleaned only while nobody uses it. Four PRs:
- **Liveness facts** (PR #12, e97455d, fc939bb):`agents/liveness.rs` decides a session's and a harness's state from injected process starts, open files and records; strict readers report each record they cannot parse.
- **The rules** (PR #18, b5e9709; the native-skip lists in 0b6d327): the eight rows with `origin`, `listed_only` for history, sealed guard rows for `.claude/sessions`, `shell-snapshots`, `plugins` and Cursor's own folders, nothing for worktrees or `memory/`; the group, switchable.
- **Session entries** (PR #21, 7455ecf..0e60c21): Claude Code's transcripts and file-history list one entry per session, with its companions, no size floor, and a `live-session` or `liveness-unknown` lock; liveness is read only when a folder holds a session. Scan 3, clean 5, progress 3, journal 5. The Review window keeps the group's folders as rows (f4a6364).
- **The clean** (PR #22, b3e7c23, 58a871a): a session entry goes to the Trash companions first, transcript last, one journal line per path; delete is `action-not-allowed`; liveness is read again before each move, and a harness folder is `agent-live` while its harness runs, holds a thread lock, or its locks cannot be read: from one process snapshot at the plan and the dry run, read afresh for each item at the guard. The Codex, opencode and Cursor rules switch from report-only to the Trash with their final hints.

**Why.** The owner (2026-10-29) wanted the coding agents' data in the Review window with a way to remove it, and chose on 2026-10-30 to rebase this story and its ADR (way A) and to send stale transcripts to the Trash after a liveness check.

**Tests.** The evidence above. Unit tier 597/597 on PR #22, `make golden-check` 13/13, DDCKit 423, the bench baseline test. The reviewer passed all nine criteria on PR #22; its one should-fix in code (the DDCKit rules fixture still marked the four harness rules without a switch) was fixed before the merge.

**Risks and follow-ups.**
- The guard reads a session's liveness from the listed entry's own files. A session's other folder (`file-history/<s>` beside `projects/<p>/<s>.jsonl`) is read by the scan, but the pruned re-check of `--items` does not read it, so a recent change or an open file only there is not seen at the plan or the guard. The record check still runs, and a live turn appends to the transcript. Follow-up: stat `file-history/<s>` from a transcript in the guard.
- Companions are listed by path and each is re-checked under the parent descriptor before its move, but the move itself goes by path, as `trash()` does today; a parent-identity check before each move would close the gap.
- The guard and the dry run sweep every process's open files once per undecided session entry; a list of dozens of sessions may add seconds. Not measured.
- `--permanent` still turns a harness folder's Trash move into a delete; the ADR forbids it only for session entries.
- Not tested at the guard: a symlink, special or dataless companion (`incomplete`); two listed sessions, one live.
- The ADR stays `proposed`: §3 is implemented, its worktree and `--agent` sections are not.

## Technical Notes

- Claude Code's own retention sweep deletes transcripts, `file-history`, `paste-cache`, `debug`, `usage-data` and the legacy folders once older than `cleanupPeriodDays` (default 30). An entry it removes between the scan and the clean is skipped `missing`. `projects/<project>/memory/` is never swept.
- `shell-snapshots` left the catalog on 2026-10-30: a live session sources its snapshot before every Bash command without holding it open, so no check sees the use, and a missing snapshot breaks the session's Bash tool. A crash leaves a few kilobytes.
- The 2026-10-18 design made `projects` a report-only region with `memory/` excluded from its bytes, and matched the `claude` process as an owner. Both are dropped: sessions are entries, and §5's in-use check plus the liveness guard replace the owner match.
- Newer Claude Code versions keep pasted images and the session scratchpad under `CLAUDE_CODE_TMPDIR`, outside `$HOME`; `ddc` does not look there.
- Tests never read the real harness folders, their records or transcripts: fake trees in a temp HOME, injected process starts and open files.

## Dependencies

- [[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]] (§3, revised 2026-10-30)
- [[epics/DDC-001/stories/story-detection-rule-catalog|Detection rule catalog]]
- [[research/coding-agent-harnesses-on-macos-where-their-data-lives-and-how-to-tell-a-live-session|Coding-agent harnesses on macOS: where their data lives and how to tell a live session]]: paths and live signals
- DDC-002 story "Review window: coding agents' data as a group": shows these rules in the Review window; waits for this one
- DDC-002 story "User data: pick your own files for the Trash": the entry model this extends

## Attachments

- Claude Code application data: https://code.claude.com/docs/en/claude-directory

---

*Last updated: 2026-10-31*

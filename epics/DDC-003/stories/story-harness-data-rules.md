---
type: story
id: "story-harness-data-rules"
epic: "DDC-003"
title: "Harness data rules"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-30
external_refs: {}
tags: [rules, agents, safety]
code_refs: ["crates/ddc-core/src/agents/"]
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Harness data rules

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | planned |
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

- [ ] Known-path rules in `rules/catalog.rs`, with `origin` and the hints of the ADR's table: `agent.claude-code-scratch` (`debug`, `todos`, `statsig`, `logs`; no `shell-snapshots`), `agent.claude-code-file-history` and `agent.claude-code-transcripts` (entries), `agent.claude-code-history`, `agent.codex-sessions`, `agent.codex-runtimes`, `agent.opencode-data` (by subfolder), `agent.cursor-agent-data`. No owner match by process name: §5's in-use check stays, and an owner match would block every path whenever Claude Code runs.
- [ ] The never-claimed list in the visitor.
- [ ] The entry model, from `ENTRY_RULES` (user data) to a per-rule entry policy: which children are entries (session ids), their companions (re-derived by name under the parent descriptor, moved transcript-last, one journal line per moved path), no size floor, Trash only; `EntryLock` `live-session` and `liveness-unknown`, replacing `recent` for these rules; the scan-report schema bump.
- [ ] The liveness guard in `exec/guard.rs`, reusing `agents::claude::sessions` (which learns to report each parse failure), `agents::codex` and `agents::running` (made `pub(crate)`), at plan time and before each move; `SkipReason::AgentLive { harness, reason }` for whole-folder items and the clean-report bump; `running::classify` learns `cursor-agent`.
- [ ] A "Coding agents" group in `GROUPS` (`rules/listing.rs`); `ddc rules --json` lists it.
- [ ] Plan gate: on the reference machine, names, sizes and times only (`ls`, `du -sk`, `stat`), never contents: which paths exist; the newer per-session folders; the Codex session layout; what `.cursor/projects` holds; with a throwaway session, whether `--resume` writes its record before opening the transcript and whether the record's `sessionId` follows `/clear`.
- [ ] `bench/make_baseline.py`: expected rule and class for the new paths.

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test (fixtures under a fake home): each path of the eight rules is found with its id, class, method and `origin`; `.local/share/opencode/auth.json` and `opencode.db` are in no finding; `every_rule_is_covered_by_a_fixture` stays green; `catalog_invariants`: no `agent.*` rule of class `safe` names `paste-cache`, `usage-data`, `file-history`, `projects` or `shell-snapshots`.
- [ ] Test: `projects/<p>/memory/MEMORY.md` and `memory/notes/x.md` are in no finding and no entry; `sessions/`, `shell-snapshots/`, `.cursor/extensions|plugins|worktrees|plans|chats` and `.codex/worktrees` are never findings; a lone `<s>/` without `<s>.jsonl` is no entry; `ide.cursor-cache` still claims the desktop app's caches; a `.venv` inside `projects/<p>/<session>/tool-results/` is not a `python.venv` finding.
- [ ] Test: a session entry (`<s>.jsonl` with `<s>/` and a `<s>.jsonl.superseded-*`) goes to the Trash as one action, its `bytes` the sum, one journal line per moved path; a companion that fails to move stops the item `failed` with `<s>.jsonl` still in place and the moved companions journaled; `--permanent` or `action: delete` on it is `action-not-allowed`; entries are listed with no size floor.
- [ ] Test (Claude Code liveness, injected process starts and open files): a fake `sessions/<pid>.json` for a live process gives that session's entry `contents.entries[].lock` `live-session` in the scan, and refuses it `entry-locked` at plan time and in the guard, while another session's entry passes; a record that appears after the plan refuses the entry in the guard; a record-less session changed under 10 minutes ago is `live-session` with reason `recent`; one unparsable record beside parsable live and stale ones, a live record with no `sessionId`, and a missing `sessions/` folder each refuse every Claude Code entry `liveness-unknown`; a record whose process is gone does not block; `paste-cache` is moved while a live session record exists.
- [ ] Test (harness liveness): a fake Codex lock whose `(dev, ino)` an injected open-files list returns, and separately an injected Codex app-server process, refuse `agent.codex-sessions` and `agent.codex-runtimes` with `agent-live`; an injected `opencode` process refuses `agent.opencode-data`; an injected `cursor-agent` process refuses `agent.cursor-agent-data`; with none running each moves.
- [ ] Test: `agent.claude-code-history` is not in a `--yes` plan, and `paste-cache` still exists after it.
- [ ] Test: `ddc rules --json` lists the group "Coding agents"; the goldens and the DDCKit fixture are updated for both schema bumps.
- [ ] `bench/make_baseline.py` expects the new rows (rule and class), and its fixture test passes. The run on the reference machine (`ddc scan ~` lists each confirmed path; `python3 -I bench/check_baseline.py`) goes to the owner's manual checks runbook at close.
- [ ] `make gate` passes.

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

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

*Last updated: 2026-10-30*

---
type: story
id: "story-review-window-coding-agents-data-as-a-group"
epic: "DDC-002"
title: "Review window: coding agents' data as a group"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-29
updated: 2026-10-31
external_refs: {}
tags: [core, app, review, agents, adr-amendment, owner-decision, user-report]
code_refs: ["app/DeepDriveCleaner/AgentsSection.swift", "app/DeepDriveCleaner/DiskView.swift", "app/Packages/DDCKit/Sources/DDCKit/AgentsSummary.swift", "app/Packages/DDCKit/Sources/DDCKit/PopoverModel.swift", "app/Packages/DDCKit/Sources/DDCKit/ReviewModel.swift", "app/Packages/DDCKit/Sources/DDCKit/ReviewRouting.swift", "app/Packages/DDCKit/Sources/DDCKit/GroupLayout.swift", "app/Packages/DDCKit/Tests/DDCKitTests/AgentsReviewTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ContractTests.swift"]
specs: []
started_at: "2026-10-30T21:16:56.390Z"
closed_at: "2026-10-30T21:42:54.061Z"
plan_updated_at: null
---

# Review window: coding agents' data as a group

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | done |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner, 2026-10-29: the Coding Agents section of the Whole disk view shows sizes and a Refresh button, nothing else. It should be in the Review window's sidebar, beside node_modules and Python environments, with a way to delete. The analysis is in the research note "Removal wherever a size is shown: coding agents, protected folders and the whole disk" (Finding 3).

**What a Review group needs.** The sidebar is built from findings, with groups from `GROUPS` in `rules/listing.rs`. So a "Coding agents" group needs catalog rules that classify the harnesses' folders. The `disk agents` report is not findings: it has no class, no plan and no re-check, and a delete from it would be a second removal path beside the safety model.

**Where the rules come from: the owner decides first** (asked on 2026-10-29). DDC-003's ADR "Agent footprint: worktrees, harness data and bounded consent for agents" §3 has them, but it is proposed, on a branch about 58 commits behind main, and it has no liveness rule for a clean and no rule for `~/.cursor`. Two ways:

- **(A) Rebase DDC-003 and amend its ADR**: liveness at clean time, `~/.cursor`, and the stale-transcript option below. One design for agents, owned by its epic; this story then shows that epic's findings.
- **(B) A narrow DDC-002 amendment of the safety ADR now**: harness rules built from `agents::HARNESSES`, reusing the liveness checks of `disk agents`. DDC-003's ADR takes them over later.

Either way the ADR work states two new semantics: a liveness refusal at plan time, and a liveness check before a move to the Trash. Today §5's in-use check runs at execution only, and a move to the Trash never counts open files (0.2.1 amendment).

**What the group would hold** (sizes from the research note "Coding-agent harnesses on macOS: where their data lives and how to tell a live session", of about 11 GB measured):

| Data | Class and action |
|---|---|
| Scratch: debug logs, todos (shell snapshots dropped, see below) | safe, deleted |
| History, file history, Codex sessions and runtimes, opencode data | review, to the Trash |
| Cursor's agent data (`~/.cursor`, about 2.8 GB): `projects`, `ai-tracking` | to be classified under (A) or (B) |
| `~/.cursor/extensions`, `plugins`, `worktrees` | never: what the user installed, and agent work on the user's repositories |
| Transcripts of stale sessions (most of Claude Code's 2.7 GB) | report only (DDC-003 as written), or review to the Trash after a liveness check: the owner's choice. `--resume` of a moved session fails until it is put back |
| A live session's files | never |
| Memory | never claimed |

The Cursor desktop app's caches under `~/Library/Application Support/Cursor` are already the rule `ide.cursor-cache` ("IDE caches"); the new rules do not claim them again.

**The group in Review.** "Coding agents", one row per harness folder kind. The sidebar has one section per class, so the group appears under Safe (scratch) and under Review (history), like any group with rules of both classes, with the class's usual checkbox, Select all and Clean. The Whole disk section stays an overview: its total counts the group's findings, and it gets a "Review…" link that opens the group. Show in Finder on its rows comes earlier, with story "Whole disk: Show in Finder and the honest action for each row".

**Amended 2026-10-30 by DDC-003.** Its ADR §3, revised for this story, is the design these rows follow: eight rules in one "Coding agents" group; sessions as entries of their project folder, to the Trash one by one; liveness at plan time and before each move (`entry-locked` with `live-session` for a session, `agent-live` for a whole harness folder); `shell-snapshots` never claimed, because a live session sources its snapshot without holding it open; opencode claimed by subfolder, never its `auth.json`. Where this table and that section differ, the section holds.

**The amended story.** "Coding agents in the Whole disk view" says of DDC-003's "Harness data rules": "not a dependency in either direction", and `ContractTests.agentsCallers` forbids a destructive call in the section's files. This story amends the first; the section itself stays non-destructive, because the delete happens in Review.

## Decomposition

- [x] The owner's decision (asked 2026-10-29): (A) or (B), and stale transcripts. — answered 2026-10-30: **(A)**, rebase DDC-003 and amend its ADR; stale transcripts are **review, to the Trash after a liveness check**.
- [x] The ADR work of the chosen way, with the plan-time liveness refusal and the liveness check before the Trash: ADR "Agent footprint: worktrees, harness data and bounded consent for agents" §3, revised for this story on 2026-10-30 (its **Liveness** paragraph amends safety ADR §5), and the dated amendment entry in the safety ADR, added at the close of "Harness data rules" (DDC-003).
- [x] Core: the harness rules (or DDC-003's, merged), with the liveness checks; a "Coding agents" group in `GROUPS`; goldens, the rules JSON and the DDCKit fixture.
- [x] App: the group in the sidebar; the section's "Review…" link; the section's total from findings. — PR #23 (89f84a6)
- [x] Vault: amend "Coding agents in the Whole disk view". — its Dependencies now point here.

## Implementation Plan

## Acceptance Criteria

- [x] Test (core): in a temp HOME with fake harness folders, the scratch and history folders are findings in "Coding agents" with the classes of the table; memory, `~/.cursor/extensions`, `plugins` and `worktrees` are never findings; `ide.cursor-cache` still claims the desktop app's caches. — evidence: DDC-003's `crates/ddc-core/tests/agent_rules.rs::each_path_of_the_eight_rules_is_found_with_its_rule_and_origin`, `::sealed_folders_and_auto_memory_are_never_findings` (memory, `.cursor/extensions`, `plugins`, `worktrees`, `ide.cursor-cache` kept)
- [x] Test (core): a finding whose harness has a live session (a fake live process or lock, as `disk agents` detects it) is refused at plan time and again before it is moved. — evidence: `crates/ddc-core/tests/harness_clean.rs::a_live_harness_keeps_its_folders` (plan, dry run and guard), `::a_record_written_after_the_plan_keeps_the_session`
- [x] Test (core): stale transcripts follow the owner's choice: no finding, or a review finding that a live session refuses. — evidence: the owner chose review, to the Trash after a liveness check: `crates/ddc-core/tests/session_entries.rs::a_live_record_or_an_open_file_locks_its_session_alone`, `harness_clean.rs::a_session_entry_goes_to_the_trash_companions_first_transcript_last`, `::a_session_whose_liveness_is_unknown_stays`
- [x] Test (DDCKit): the group's place in the sidebar and its labels; the section's link; `ContractTests.agentsCallers` still passes. — evidence: `AgentsReviewTests.label`, `.sidebar`, `.selectAndClean`, `.linkKeepsSelection`, `.linkOpensFresh`, `.linkLeavesWindowAlone`, `.linkRequest`, `.total`, `.popoverTotal`, `.noTotal`; `ContractTests.agentsCallers`
- [ ] Manual (owner, moved to the owner-checks runbook at close, 2026-10-31): the Review window shows "Coding agents" with sizes; cleaning a scratch row frees it; a live harness's folder is skipped with "<Harness> is running".

## Final Summary

**What changed.** The coding agents' data has its own group in the Review window, and the Whole disk view leads to it.
- The core is story "Harness data rules" (DDC-003), the way (A) the owner chose on 2026-10-30: eight rules in a "Coding agents" group, sessions as entries, liveness at the plan and before each move, stale transcripts to the Trash after the check.
- In the app the group appears under Safe and under Review like any group; its rows select and clean through the usual path. Claude Code's transcripts and file-history rows are shown but locked "Manual only"; their sessions can be moved only with `ddc clean --items` (a session folder stays one row: DDC-003's f4a6364). The Review… link also sits in the window's own Whole disk detail.
- The Whole disk view's Coding Agents section says "In Review: <size> in <n> folders", from the last scan's findings in the group, counted as the sidebar counts them; nothing for a missing or foreign report. Its **Review…** link opens the Review window on the group; with the window open it keeps the person's selection (PR #23, 89f84a6).
- The section stays non-destructive: `ContractTests.agentsCallers` now counts the link and holds DiskView to one `.review(` call.

**Why.** The owner (2026-10-29): the Coding Agents section showed sizes and a Refresh button, nothing else; the data should sit in Review beside node_modules, with a way to remove it.

**Tests.** The evidence above. DDCKit 433 tests on PR #23; the unsigned `xcodebuild` build passed before it; its Gate's required jobs were green. The implementing agent ran the reviewer before its commit and applied its should-fixes (the selection kept only while the window is open, the total computed in PopoverModel).

**Risks and follow-ups.**
- The owner's chosen outcome for stale transcripts (to the Trash from Review) is not reachable in the window yet: the transcripts and file-history rows are locked "Manual only: ddc has no command for it", which is also wrong for them. Follow-up: story "Review window: pick a coding agent's sessions one by one" (DDC-002), next after 0.11.0.
- The section's total counts findings, the rest of its sizes come from `ddc disk agents`; the two can differ (different paths, different times).
- The manual check is in the owner-checks runbook; the app part ships in 0.11.0.

## Technical Notes

- Tests never read the real harness folders or their databases; fake trees in a temp HOME only.

## Dependencies

- The owner's decision between (A) and (B)
- Research note "Removal wherever a size is shown: coding agents, protected folders and the whole disk" (Finding 3, open questions 2 and 3)
- Research note "Coding-agent harnesses on macOS: where their data lives and how to tell a live session"
- DDC-003, on main (PRs #12, #18, #21, #22): ADR "Agent footprint: worktrees, harness data and bounded consent for agents", story "Harness data rules"
- "Coding agents in the Whole disk view" (DDC-002): amended by this story; closed 2026-10-30

## Attachments

-

---

*Last updated: 2026-10-31*

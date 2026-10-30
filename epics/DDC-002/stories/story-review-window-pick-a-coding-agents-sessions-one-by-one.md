---
type: story
id: "story-review-window-pick-a-coding-agents-sessions-one-by-one"
epic: "DDC-002"
title: "Review window: pick a coding agent's sessions one by one"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-31
updated: 2026-10-31
external_refs: {}
tags: [app, review, agents, entries, follow-up]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Review window: pick a coding agent's sessions one by one

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The follow-up of story "Review window: coding agents' data as a group" (DDC-002). The owner chose on 2026-10-30 that stale Claude Code transcripts go to the Trash from Review after a liveness check. The core does it: story "Harness data rules" (DDC-003) lists each session as an entry of its folder, with a `live-session` or `liveness-unknown` lock, and `ddc clean --items` moves a listed session's transcript and companions to the Trash after checking liveness again. The window does not offer it yet:

- The Claude Code transcripts and file-history rows are locked `.noAction`, "Manual only: ddc has no command for it" (`ReviewItems.swift`). That text is wrong for them: ddc has a command, for their entries.
- `GroupLayout` lists entries one by one only in a user-data folder, and `ReviewItems.entryRows` builds entry rows only for user-data folders, so a session folder stays one locked row.

**What the window does.** Review → Coding agents mixes seven rules, so it keeps the rules shape: every other rule's row stays listed and in Select all. The two session folders (a project's transcripts, the file-history folder) become subgroups whose items are their sessions:

- A session row shows its id, its size with its companions and its newest change, and its lock in the existing wording (`CleanWording.entryLock`): "A running Claude Code has this session open", "The session changed in the last 10 minutes", "A running process holds a file of this session open", or "Could not tell whether the session is in use · <cause>". A locked session cannot be picked. No user-data facts ("A project (has .git)") and no "Protect this folder" on a session.
- Sessions are picked one by one: never in Select all. The folder heading is not checkable, and the folder row is locked with a new reason ("Pick its sessions one by one below") instead of "Manual only"; it is never selectable and never sent in `--items`.
- Session rows sit under Review, by their folder's class, while their payload keeps the class the core requires.
- Picked sessions go to `ddc clean --items` as `class: user-data, action: trash`, never `permanent`: the core takes a session entry only so (`session_entries.rs::the_plan_the_dry_run_and_the_clean_refuse_a_session_entry_listed_as_review`, `harness_clean.rs::a_session_entry_is_never_deleted`). The sheet offers no Delete permanently for them, and its class row does not call them "User data".
- The confirmation takes each folder's rule hint: for a transcript, `--resume` of it fails until it is put back; for file history, /rewind cannot use it once moved.
- Session entries stay out of the Items view, as user-data entries do.

## Decomposition

- [ ] DDCKit: session subgroups inside the rules shape of Review → Coding agents (not the entries layout, which would hide the other rules: `PickedMoveTests.filtered` holds "never both rows" and is revised on purpose); session rows under Review; the folder's new lock in place of `.noAction`.
- [ ] DDCKit: the picked sessions go to `--items` as `user-data`/`trash`; the confirmation with each folder's hint, no Delete permanently, no "User data" class row; no Protect on a session.
- [ ] App: whatever the subgroup view needs for these rows.
- [ ] Vault, at close: revise the runbook lines that say the window keeps a session folder as one row ("Harness data rules", DDC-003) and that the rows are locked "Manual only" ("Review window: coding agents' data as a group", DDC-002).

## Implementation Plan

## Acceptance Criteria

- [ ] Test (DDCKit): from a report whose transcripts folder lists three sessions, one `live-session`, one `liveness-unknown`, one free, the Coding agents group under Review lists the three under their folder with the existing lock wording; only the free one can be picked; the other rules' rows stay listed and in Select all; no session is in Select all; the folder row is never selectable.
- [ ] Test (DDCKit): picking a transcript session and a file-history session and cleaning sends `clean --items` with exactly those two items, each `class: user-data, action: trash`; the sheet shows no Delete permanently and names each folder's hint (`--resume`, /rewind).
- [ ] Test (DDCKit): no row of the group reads "Manual only: ddc has no command for it"; a session offers no Protect; `ContractTests` still hold.
- [ ] Manual (owner): pick one stale Claude Code session in the Review window and clean it: its transcript and companions are in the Trash; after Put Back, `claude --resume` of it works. A running session cannot be picked.

## Final Summary

## Technical Notes

- The user-data entries path (`ReviewItems.entryRows`, `GroupLayout.entriesLayout`, the picked-entries clean) is the model; the difference is the lock set and the wording.
- The core needs no change: plan, dry run and guard already take a session entry under the listed consent.

## Dependencies

- "Review window: coding agents' data as a group" (DDC-002): the group this extends.
- "Harness data rules" (DDC-003): the session entries, their locks and the clean.

## Attachments

-

---

*Last updated: 2026-10-31*

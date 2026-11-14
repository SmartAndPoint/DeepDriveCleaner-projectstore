---
type: story
id: "story-ddc-agent-clean-session-clean-up-after-one-session"
epic: "DDC-003"
title: "ddc agent clean --session: clean up after one session"
status: planned
priority: p3
assignee: "Evgenii Konev"
created: 2026-11-14
updated: 2026-11-14
external_refs: {}
tags: [cli, agents, cleanup, consent]
code_refs: []
specs: []
# no_spec: "<why this story needs no spec>"  (written at the plan gate)
# variant: test  (a test story; its guide is story.test)
started_at: null
closed_at: null
plan_updated_at: null
---

# ddc agent clean --session: clean up after one session

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | planned |
| **Priority** | p3 |
| **Assignee** | Evgenii Konev |

---

## Description

`ddc agent clean --session <id>` narrows the agent's clean to the `safe` items inside one session's worktree: the default a skill uses to clean up after itself (ADR "Agent footprint", §7). The worktree itself and the session's temporary folder are listed for the person, never removed by the agent.

## Decomposition

- [ ] Map a session id to its worktree (the session record's working folder, or the worktree lock's pid) through the per-session footprint's reading
- [ ] `--session <id>` on `ddc agent clean`, its refusals (an unknown session, a session outside ROOT's project), README

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test: with two sessions in two worktrees of one project, `--session <a>` removes only the safe items inside a's worktree (opt-in on) and lists a's worktree and temporary folder for the person; an unknown id exits 2

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- Split out of story "Clean: --agent bounded consent" at its plan gate (2026-11-14, the owner's pick). It depends on story "Per-session footprint: what each agent session keeps in its transcripts, temporary folders and worktrees" for the session → worktree mapping.

## Dependencies

- [[epics/DDC-003/stories/story-clean-agent-bounded-consent|Clean: --agent bounded consent]]: blocking
- [[epics/DDC-003/stories/story-per-session-footprint-transcripts-temporary-folders-and-worktrees|Per-session footprint: what each agent session keeps in its transcripts, temporary folders and worktrees]]: blocking (session → worktree)

## Attachments

-

---

*Last updated: 2026-11-14*

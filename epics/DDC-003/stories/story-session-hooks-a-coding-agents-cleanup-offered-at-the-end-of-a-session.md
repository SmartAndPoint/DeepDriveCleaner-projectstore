---
type: story
id: "story-session-hooks-a-coding-agents-cleanup-offered-at-the-end-of-a-session"
epic: "DDC-003"
title: "Session hooks: a coding agent's cleanup offered at the end of a session"
status: planned
priority: p3
assignee: "Evgenii Konev"
created: 2026-11-18
updated: 2026-11-18
external_refs: {}
tags: ["agents", "hooks", "claude-code", "qwen", "antigravity"]
code_refs: []
specs: []
# no_spec: "<why this story needs no spec>"  (written at the plan gate)
# variant: test  (a test story; its guide is story.test)
started_at: null
closed_at: null
plan_updated_at: null
---

# Session hooks: a coding agent's cleanup offered at the end of a session

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

ADR "Agent footprint" §7 allows a session-end hook that shows the person what `ddc agent clean` lists for them and removes what the person approves, through `ddc clean --items`; the person's yes must be the person's, in a sheet or a terminal, never only in the agent's chat. Harnesses differ, and none has a session-end hook that asks: Claude Code's `SessionEnd` only shows its stderr, within a short time budget, and `ask` exists only in `PreToolUse`, around a tool call the model makes; Qwen Code asks only in its TUI; Antigravity has `ask` but no session-end event, only `Stop`; Codex and Gemini CLI document no hook that waits for a person; opencode's plugins can ask through its permission API. The owner asked whether hooks can work everywhere (2026-11-17).

**A harness's own permission dialog is not the person's consent of §7** until the owner decides otherwise: it can be approved once for all ("Always Allow", bypass modes), and `ddc clean --items` takes its list as consent and asks nothing more, which would make an unattended clean, parked by the owner on 2026-10-28. So a hook here shows the person the list and runs nothing that removes; the person cleans in the app's sheet or their own terminal.

This story decides per harness what a hook can show, and ships the first one, most likely Claude Code's, carried by its plugin.

## Decomposition

Rough until the plan gate.

- [ ] Per harness, from its own documentation: which event, whether it can ask the person and wait, its time budget, and how a hook is installed; opencode's plugin events among them
- [ ] With the owner: whether a harness's permission dialog may ever count as consent (default no), and whether a hook may open the app's Review window
- [ ] The first hook, in the Claude Code plugin, with a test of what it runs
- [ ] The guidance says what the hook does and that it never removes without the person's answer

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

- [ ] Review: each shipped hook removes nothing without an answer the person gives outside the agent's chat (ADR §7)
- [ ] Test: every command a shipped hook runs is read-only (`ddc agent clean --dry-run`, `ddc agent footprint`), and none is `ddc clean`
- [ ] Owner check (runbook): at the end of a Claude Code session the hook shows what is left for the person, and nothing is removed

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- From the research pass of 2026-11-17 (to read first-hand at the plan gate): Claude Code hooks https://code.claude.com/docs/en/hooks; Codex hooks https://learn.chatgpt.com/docs/hooks (`ask` in `PreToolUse` "parsed but not supported yet"); Gemini CLI https://geminicli.com/docs/hooks/reference/ (only `allow` and `deny`); Qwen Code https://qwenlm.github.io/qwen-code-docs/en/users/features/hooks/ (`ask` in the TUI, `deny` headless); Antigravity https://www.antigravity.google/docs/hooks/ (`ask` respects "Always Allow"; no session-end event); opencode plugins https://opencode.ai/docs/plugins/.
- Claude Code's `SessionEnd` runs in a short budget (about 1.5 s by default, longer with its `timeout`): `ddc agent clean --dry-run` takes seconds on a large project.

## Dependencies

- [[epics/DDC-003/stories/story-agent-guidance-skill-and-agents-snippet|Agent guidance: skill and AGENTS.md snippet]] (done) and [[epics/DDC-003/stories/story-agent-skill-install-the-shared-agentsskills-folder|Agent skill install: the shared ~/.agents/skills folder]]
- [[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]] §7
- [[epics/DDC-003/stories/story-claude-code-plugin-ddcs-skill-from-a-marketplace-in-this-repository|Claude Code plugin: ddc's skill from a marketplace in this repository]]

## Attachments

-

---

*Last updated: 2026-11-18*

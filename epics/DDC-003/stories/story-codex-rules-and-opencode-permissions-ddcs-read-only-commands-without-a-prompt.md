---
type: story
id: "story-codex-rules-and-opencode-permissions-ddcs-read-only-commands-without-a-prompt"
epic: "DDC-003"
title: "Codex rules and opencode permissions: ddc's read-only commands without a prompt"
status: planned
priority: p3
assignee: "Evgenii Konev"
created: 2026-11-18
updated: 2026-11-18
external_refs: {}
tags: ["agents", "codex", "opencode", "permissions"]
code_refs: []
specs: []
# no_spec: "<why this story needs no spec>"  (written at the plan gate)
# variant: test  (a test story; its guide is story.test)
started_at: null
closed_at: null
plan_updated_at: null
---

# Codex rules and opencode permissions: ddc's read-only commands without a prompt

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

In Claude Code the skill's `allowed-tools` lets the read-only `ddc` commands run without a prompt (ADR "Agent footprint" §7, note of 2026-11-17: `ddc changes`, `ddc agent status`, `ddc agent footprint`, `ddc agent clean --dry-run`, `ddc rules`). Codex and opencode ignore that field: each read-only call is a prompt there. Codex has rule files (`~/.codex/rules/*.rules`, `prefix_rule(pattern=[...], decision="allow"|"prompt"|"forbidden")`), and opencode a `permission` block in its config (`{"action": "shell", "resource": "git status *", "effect": "allow"}`, the last match wins). The owner works in both (2026-11-17).

This story decides whether `ddc agent skill install` writes such a rule or block (into files the person owns, so with the same consent and marker care as the skill), or the guidance shows the person the lines to add.

## Decomposition

Rough until the plan gate.

- [ ] Read Codex's rules page and opencode's permissions page first-hand
- [ ] Decide with the owner: written by `install` (a new file, never an edit of the person's config without the same block rules as `AGENTS.md`), or shown for the person to add
- [ ] Whatever is shipped names exactly the five read-only commands, never `ddc agent clean` without `--dry-run` or `ddc agent skill install`
- [ ] A try-out in Codex and opencode

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

- [ ] Test: the rule or block names exactly the five read-only commands under both binary names, and no removal
- [ ] Owner check (runbook): in Codex and opencode, `ddc changes --json` runs without a prompt and `ddc agent clean` still asks

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- Codex rules: https://learn.chatgpt.com/docs/agent-configuration/rules (verified by the critic, 2026-11-18). opencode permissions: https://opencode.ai/docs/permissions/ (from the research pass; the docs describe a V2 that renames V1's keys).

## Dependencies

- [[epics/DDC-003/stories/story-agent-skill-install-the-shared-agentsskills-folder|Agent skill install: the shared ~/.agents/skills folder]]
- [[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]] §7 (the allowed-tools note of 2026-11-17)

## Attachments

-

---

*Last updated: 2026-11-18*

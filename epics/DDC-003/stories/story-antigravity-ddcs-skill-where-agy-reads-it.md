---
type: story
id: "story-antigravity-ddcs-skill-where-agy-reads-it"
epic: "DDC-003"
title: "Antigravity: ddc's skill where agy reads it"
status: planned
priority: p3
assignee: "Evgenii Konev"
created: 2026-11-18
updated: 2026-11-18
external_refs: {}
tags: ["agents", "skill", "antigravity"]
code_refs: []
specs: []
# no_spec: "<why this story needs no spec>"  (written at the plan gate)
# variant: test  (a test story; its guide is story.test)
started_at: null
closed_at: null
plan_updated_at: null
---

# Antigravity: ddc's skill where agy reads it

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

Google Antigravity (the IDE and its CLI, `agy`) reads Agent Skills from `.agents/skills/` in a project, so a project install already reaches it; its global folder is `~/.gemini/config/skills/` (the CLI also `~/.gemini/skills/`), which `ddc agent skill install` does not write. Whether it reads `~/.agents/skills` is not documented. It also imports Claude Code plugins (`agy plugin import claude`) and installs plugins from a GitHub URL. The owner uses Antigravity (2026-11-17), and Gemini CLI is moving to it.

The plan gate decides between three ways: a third family for `install` (`~/.gemini/config/skills/ddc`), which also means sealing that folder, a new harness in `rules/catalog/shared/agents.rs` (there is no Gemini or Antigravity rule yet); `agy plugin import claude`, which reads `~/.claude/plugins/` and so needs the Claude Code plugin installed on the same machine; or an Antigravity plugin manifest in this repository for `agy plugin install https://github.com/...`.

## Decomposition

Rough until the plan gate.

- [ ] Read Antigravity's skills, rules and plugin pages first-hand; check whether it reads `~/.agents/skills`
- [ ] Either a family `antigravity` in `ddc agent skill install`, sealed like the others, or the guidance's note on `agy plugin import claude`
- [ ] A try-out on the owner's machine

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

- [ ] Test (if a family is added): `install` and `status` cover Antigravity's folder like the other two, and a scan never offers it
- [ ] Owner check (runbook): an Antigravity session lists the `ddc` skill and uses it when asked why the disk is full

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- From the research pass of 2026-11-17, not yet read first-hand: https://www.antigravity.google/docs/skills/ (project `.agents/skills/<name>/`, global `~/.gemini/config/skills/<name>/`, only `description` required), https://www.antigravity.google/docs/plugins (`agy plugin install <github-url>`, `agy plugin import claude`), https://www.antigravity.google/docs/rules/ (`.agents/rules/*.md` need a `trigger` frontmatter).
- Gemini CLI's page says it was replaced by Antigravity CLI on 2026-06-18 for free and Google One users (https://geminicli.com/docs/extensions/reference/).

## Dependencies

- [[epics/DDC-003/stories/story-agent-guidance-skill-and-agents-snippet|Agent guidance: skill and AGENTS.md snippet]] (done) and [[epics/DDC-003/stories/story-agent-skill-install-the-shared-agentsskills-folder|Agent skill install: the shared ~/.agents/skills folder]]
- [[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]] §7

## Attachments

-

---

*Last updated: 2026-11-18*

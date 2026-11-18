---
type: story
id: "story-claude-code-plugin-ddcs-skill-from-a-marketplace-in-this-repository"
epic: "DDC-003"
title: "Claude Code plugin: ddc's skill from a marketplace in this repository"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-18
updated: 2026-11-18
external_refs: {}
tags: ["agents", "skill", "plugin", "claude-code"]
code_refs: []
specs: []
# no_spec: "<why this story needs no spec>"  (written at the plan gate)
# variant: test  (a test story; its guide is story.test)
started_at: null
closed_at: null
plan_updated_at: null
---

# Claude Code plugin: ddc's skill from a marketplace in this repository

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

ADR "Agent footprint" §7 makes the skill a plugin of a SmartAndPoint Claude Code marketplace too; the story "Agent guidance: skill and AGENTS.md snippet" deferred it (D1, the owner, 2026-11-17), since no such marketplace existed. A plugin lets a person install and update the skill with `/plugin install`, and later carry hooks. The owner asked for one story per harness (2026-11-17); this is Claude Code's.

The open questions for the plan gate: where the marketplace lives (this repository's `.claude-plugin/marketplace.json`, or a SmartAndPoint repository of its own, which is a public change and the owner's call); how the plugin's skill stays the same bytes as `agents/claude-code/ddc/SKILL.md` without moving it (one way: the plugin's root is `agents/claude-code/`, its manifest `agents/claude-code/.claude-plugin/plugin.json` inside the MIT paths, its skills `["."]`, so there is no copy); whether the manifest carries a `version` at all, since a set `version` keeps every user on the cached copy until it changes and without one the commit decides; and what `ddc agent skill status` says when both the plugin and an installed copy are there (ADR §7: keep the plugin). Qwen Code converts Claude plugins (`qwen extensions install`); whether Codex reads a Claude marketplace is unverified: check both.

## Decomposition

Rough until the plan gate.

- [ ] The marketplace's place and its manifest, decided with the owner
- [ ] The plugin: `plugin.json`, its `skills/ddc/SKILL.md` kept equal to `agents/` by a test
- [ ] The manifest's `version`: none (the commit decides), or written by `make bump` every release
- [ ] `ddc agent skill status` names a plugin copy beside an installed one, or the guidance says to keep one
- [ ] A try-out: `/plugin marketplace add` and `/plugin install` on the owner's machine, and whether Codex and Qwen Code take the same plugin

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

- [ ] Test: the plugin's skill is byte for byte `agents/claude-code/ddc/SKILL.md`
- [ ] Test: the marketplace and plugin manifests parse and name the plugin and its version
- [ ] Owner check (runbook): the plugin installs from the marketplace in Claude Code, and its skill is the one listed

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- Claude Code: a plugin is `.claude-plugin/plugin.json` (only `name` required) with `skills/`, `hooks/hooks.json`, `commands/`; a marketplace is `.claude-plugin/marketplace.json` with `name`, `owner` and `plugins[]`; names starting with `claude-` or `anthropic-` are refused; installed with `/plugin marketplace add owner/repo` then `/plugin install name@marketplace`; third-party marketplaces do not update on their own by default (https://code.claude.com/docs/en/plugins-reference, https://code.claude.com/docs/en/plugins/marketplace-reference; from the research pass of 2026-11-17, to be read first-hand at the plan gate).
- The MIT block of the README names `agents/**` and `docs/*.schema.json` only (`the_licences_follow_their_paths`): a manifest at the repository root needs that block or the test to change.

## Dependencies

- [[epics/DDC-003/stories/story-agent-guidance-skill-and-agents-snippet|Agent guidance: skill and AGENTS.md snippet]] (done) and [[epics/DDC-003/stories/story-agent-skill-install-the-shared-agentsskills-folder|Agent skill install: the shared ~/.agents/skills folder]]
- [[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]] §7

## Attachments

-

---

*Last updated: 2026-11-18*

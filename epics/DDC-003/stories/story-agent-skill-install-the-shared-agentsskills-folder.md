---
type: story
id: "story-agent-skill-install-the-shared-agentsskills-folder"
epic: "DDC-003"
title: "Agent skill install: the shared ~/.agents/skills folder"
status: in-progress
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-18
updated: 2026-11-18
external_refs: {}
tags: [agents, skill, codex, opencode, gemini, qwen, cli]
code_refs: []
specs: ["agent-skill-install"]
started_at: "2026-11-18T06:17:42.156Z"
closed_at: null
plan_updated_at: "2026-11-18T06:17:42.156Z"
status_changed_at: "2026-11-18T06:17:42.156Z"
---

# Agent skill install: the shared ~/.agents/skills folder

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | in-progress |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

`ddc agent skill install` (0.38.0, story "Agent guidance: skill and AGENTS.md snippet") writes the skill only where Claude Code reads it, `~/.claude/skills/ddc`. The owner uses Codex, opencode and Google Antigravity too, and asked for the guidance to work in every harness (2026-11-17). A research pass, then the vendors' own pages for two of them, found one shared folder: `~/.agents/skills/` (and `.agents/skills/` in a project), read by Codex (from the current folder up to the repository root, and `$HOME/.agents/skills`), opencode (beside `~/.claude/skills`), Gemini CLI and Qwen Code. The skill's format, `SKILL.md` with `name` and `description`, is the same. The owner chose to ship this before 0.38.0 (2026-11-18).

So `install` writes the same skill into `~/.agents/skills/ddc` too, and `status` reports that folder. Antigravity's global folder, the Claude Code plugin and hooks are later stories.

**Independent test:** in a synthetic home, `ddc agent skill install --approved` leaves `SKILL.md` and `.ddc-skill` in both `~/.claude/skills/ddc` and `~/.agents/skills/ddc`, and `ddc agent skill status --json` lists both as `current`.

## Decomposition

- [ ] PR 0, vault: this story, its plan, the spec's amendment and the ADR's dated note
- [ ] PR 1, step 1: the families in the core (`guidance::Family`, its folder under a root) and `family` in each folder's status
- [ ] PR 1, step 2: `--family` and the all-or-nothing plan in `install`; `status` over both families
- [ ] PR 1, step 3: the rule `agent.shared-skills-sealed`, with its group, the app's wording and the rule tests
- [ ] PR 1, step 4: `docs/skill.schema.json` regenerated, the cases, the macOS golden `rules.json`
- [ ] PR 1, step 5: the guidance and both READMEs say where the skill lands
- [ ] The owner's try-out in Codex and opencode goes to the runbook at close

## Implementation Plan

A thin route through spec "Agent skill install", contracts 1 and 9 (amended in PR 0); every other contract applies to the new folder unchanged, since it is a skill folder like the first.

- D1. **Both by default.** `install` writes the skill folder of each harness family it knows: `claude-code` (`.claude/skills/ddc`) and `agents` (`.agents/skills/ddc`, the shared one), under the home or under `--project DIR`. `--family claude-code|agents`, repeatable, narrows it. The default is both because 0.38.0 is not released yet, so no shipped behaviour changes, and a person who asked for the guidance wants it where their agents read it; the question lists every file before anything is written.
- D2. **One plan, all or nothing.** Every folder's plan is made before the question, and a refusal in any (a link, an edited copy) refuses the whole install, as an `AGENTS.md` refusal does today.
- D3. **Status** reports every folder of both families, the home's and with `--project` the project's, in `skills[]`, each with a new required `family` field (`claude-code`, `agents`). Not `harness`: the scan report's `origin.harness` names harnesses (`claude-code`, `codex`, …), and `agents` is a folder four of them share (critic, 2026-11-18). `docs/skill.schema.json` stays version 1, as additive fields have before (`grew`, `read_at`); 0.38.0, its first release, is not out.
- D4. **Sealed.** `.agents/skills` is never claimed, by a rule of its own, `agent.shared-skills-sealed` (user data, no removal, not listed, in the group "Coding agents"), not inside Claude Code's sealed rule: the folder is no harness's own, and a rule id is costly to rename once released (critic, 2026-11-18).
- D5. **Twice in opencode.** opencode reads both `~/.claude/skills` and `~/.agents/skills`; the skill's path-derived ID is the same (`ddc`), so it loads one, and both copies are the same bytes. Codex does not read `~/.claude/skills`, so it sees one.

Files: `crates/ddc-core/src/guidance/mod.rs` (the families), `crates/ddc-cli/src/cli.rs` and `cmd/agent_skill.rs`, `crates/ddc-core/src/rules/catalog/shared/agents.rs` and the catalogs, `rules/listing.rs`, DDCKit's `CleanWording.swift` and `ReviewItems.swift`, the tests (`guidance.rs`, `agent_skill_cli.rs`, `agent_rules.rs`, `rules.rs`, `rules_cli.rs`, cases), `docs/skill.schema.json`, `bench/golden/rules.json`, `agents/claude-code/ddc/SKILL.md`, `agents/AGENTS.snippet.md`, `agents/README.md`, `README.md`.

## Acceptance Criteria

- [ ] Test: by default `install` writes `SKILL.md` and the marker into both `.claude/skills/ddc` and `.agents/skills/ddc` (home, and a project with `--project`); `--family agents` writes only the shared one
- [ ] Test: an edited or linked shared folder refuses the whole install, and nothing is written in either folder
- [ ] Test: `status --json` lists both folders with their `family` and state; the committed schema is current
- [ ] Test (`agent_rules.rs`): a scan of a home with `~/.agents/skills/ddc/` has no finding at or under `~/.agents/skills`
- [ ] The acceptance of spec "Agent skill install" still holds, with contracts 1 and 9 as amended
- [ ] The PR's Gate is green, the Linux and native Windows legs included
- [ ] Owner check (runbook "Owner checks: manual acceptance across stories"): after `ddc agent skill install`, a Codex session and an opencode session each list the `ddc` skill and use it when asked why the disk is full

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Status log

- 2026-11-18T06:17:42.156Z status: planned → in-progress — work started (plan gate)

## Technical Notes

- Sources, read 2026-11-18: Codex, https://learn.chatgpt.com/docs/build-skills (`$HOME/.agents/skills`, `.agents/skills` up to the repository root; `name` and `description` required; same-named skills are not merged, so a skill in both the home and a project shows twice); opencode, https://opencode.ai/docs/skills/ (`~/.agents/skills` beside `~/.claude/skills` and `~/.config/opencode/skills`; a later source with the same ID wins); Gemini CLI, https://geminicli.com/docs/cli/skills/ (`~/.agents/skills` an alias of `~/.gemini/skills`, workspace over user); Qwen Code, https://qwenlm.github.io/qwen-code-docs/en/users/features/skills/ (`~/.agents/skills` and `.agents/skills` without any configuration; `~/.qwen/skills` wins a name clash). The last two read by the critic.
- The plan is all or nothing; the writes are one folder after another. An I/O error in the second folder leaves the first installed: `status` shows it, and the next `install` finishes it. A `~/.agents` that is a file gives a plain I/O error (exit 1), not a named refusal.
- The skill's `allowed-tools` is Claude Code's field; Codex and opencode ignore fields they do not know.

## Dependencies

- [[epics/DDC-003/stories/story-agent-guidance-skill-and-agents-snippet|Agent guidance: skill and AGENTS.md snippet]] (done): the install this story widens
- [[specs/agent-skill-install|Agent skill install]]: contracts 1 and 9

## Attachments

-

---

*Last updated: 2026-11-18*

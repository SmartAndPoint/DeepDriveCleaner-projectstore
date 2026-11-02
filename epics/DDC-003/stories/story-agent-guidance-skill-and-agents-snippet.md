---
type: story
id: "story-agent-guidance-skill-and-agents-snippet"
epic: "DDC-003"
title: "Agent guidance: skill and AGENTS.md snippet"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-11-02
external_refs: {}
tags: [agents, docs, skill]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Agent guidance: skill and AGENTS.md snippet

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

*Amended 2026-11-02 (ADR "Agent footprint", §7): `ddc agent skill install` / `status` install and report the skill and the snippet; the skill is also a plugin of the SmartAndPoint marketplace; the guidance names `ddc agent status`, `ddc agent footprint` and `ddc agent clean`, and proposes installing `ddc` only after the person agrees.*

The code repository gets an `agents/` folder with what a coding agent needs to use `ddc` well: §6 of the ADR "Agent footprint: worktrees, harness data and bounded consent for agents". A Claude Code skill (`agents/claude-code/ddc/SKILL.md`) and an `AGENTS.md` snippet (`agents/AGENTS.snippet.md`, for Codex, opencode and any harness that reads `AGENTS.md`) say when to run `ddc footprint`, what `ddc clean --agent` may and may not do, how to read the two JSON contracts, and that a worktree is removed through `ddc`, never through the agent's own `git worktree remove`. The guidance decides nothing: every rule it states cites the ADR section or the schema field it comes from.

The MCP server is deferred by the ADR; this story does not start it.

## Decomposition

Rough until the plan gate.

- [ ] `agents/claude-code/ddc/SKILL.md`: frontmatter `name: ddc`, a `description` that triggers on disk space, worktrees, environments and "clean up after yourself"; body in four parts: when (session start in a worktree, before installing dependencies into a new worktree, on `ENOSPC` or a failed install, at the end of a session), what to run (`ddc footprint --json`, then `ddc clean --agent --project <main> --json`), how to read the result (the `state` fields, `reclaimable_now`, `not-consented`), what not to do (no `--yes`, no `--items` without `--agent`, no `git worktree remove --force`, no `rm -rf` of a finding)
- [ ] `agents/AGENTS.snippet.md`: the same in the plain prose an `AGENTS.md` carries, with markers so a project can paste and later replace it
- [ ] `agents/README.md`: how to install the skill (copy into `.claude/skills/` or `~/.claude/skills/`) and the snippet; a note that the skill is for the repository where an agent works, not for the `ddc` repository
- [ ] The two JSON contracts are cited by field name; a test keeps the skill's field names in step with `docs/footprint.schema.json` and `docs/clean.schema.json` (a structure test in `crates/ddc-cli/tests/structure.rs`, which already checks repository layout)
- [ ] README of the code repository: an "For coding agents" section pointing to `agents/`
- [ ] Try it: on the reference machine, a Claude Code session with the skill installed, in a worktree with a fresh checkout, asked to set up the project; read the transcript for the `footprint` call before the install

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test (`structure.rs`): `agents/claude-code/ddc/SKILL.md` and `agents/AGENTS.snippet.md` exist, the skill's frontmatter has `name` and `description`, and every backticked `ddc` command in both files is accepted by the CLI's parser (`clap`'s `try_get_matches_from` over each command line) — no documented flag that does not exist
- [ ] Test: every JSON field the two files name in backticks exists in `docs/footprint.schema.json` or `docs/clean.schema.json`
- [ ] Review: neither file contains an instruction the ADR does not state (checked by the reviewer against §4, §5 and §6 of the ADR, listed in the final summary)
- [ ] On the reference machine: a Claude Code session with the skill installed, started in a fresh worktree and asked to install the project's dependencies, runs `ddc footprint` before the install; a second session asked to "free some space in this project" runs `ddc clean --agent --project <main>` and nothing else that removes files; both transcripts cited in the final summary
- [ ] `make check` passes

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- Claude Code loads skills from `.claude/skills/<name>/SKILL.md` in the project and `~/.claude/skills/` for the user; a skill's `description` is what decides whether it is used. The frontmatter is the only machine-read part; the body is instructions.
- The `AGENTS.md` snippet uses the ProjectStore marker style (`<!-- ddc:agents … -->` … `<!-- /ddc:agents -->`) so a later `ddc` version can replace it in place; replacing is not in this story.
- A skill that says "run `ddc clean --agent`" is only as safe as `--agent` itself; the guidance adds no permission and the test that commands parse keeps it from drifting from the CLI.
- The "try it" check is one transcript read, not an evaluation suite; whether the skill triggers reliably is a later question, once the skill has a few weeks of use.

## Dependencies

- [[epics/DDC-003/stories/story-ddc-footprint-scoped-project-report|ddc footprint: a scoped project report]]
- [[epics/DDC-003/stories/story-clean-agent-bounded-consent|Clean: --agent bounded consent]]
- [[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]]

## Attachments

- Claude Code skills: https://code.claude.com/docs/en/skills

---

*Last updated: 2026-10-18*

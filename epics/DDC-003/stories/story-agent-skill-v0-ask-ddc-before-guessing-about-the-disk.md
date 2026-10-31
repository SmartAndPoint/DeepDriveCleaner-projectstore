---
type: story
id: "story-agent-skill-v0-ask-ddc-before-guessing-about-the-disk"
epic: "DDC-003"
title: "Agent skill v0: ask ddc before guessing about the disk"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-31
updated: 2026-10-31
external_refs: {}
tags: [agents, skill, agents-md, docs]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Agent skill v0: ask ddc before guessing about the disk

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner wants agents themselves to know about disk usage: "a skill for agents" (2026-10-31). The full guidance story, "Agent guidance: skill and AGENTS snippet", is built on `ddc footprint` and `ddc clean --agent`, which do not exist yet. This v0 ships what agents can use today: read-only questions with answers in JSON. It never lets an agent clean.

What the skill teaches an agent:

- **When to ask `ddc`.**
  - The person asks about disk space, or a command fails with `ENOSPC` or "No space left on device".
  - Before installing dependencies into a new worktree or a fresh checkout.
  - When the person asks what the agent's own sessions left behind.
- **What to run**, all read-only:
  - `ddc disk agents --json`: harnesses, live sessions and, once released, the per-session footprint;
  - `ddc scan <dir> --json`: what can be freed under a project;
  - `ddc rules --json`: what each rule means.
- **How to read the answer.** It cites the schema fields by name: `docs/agents.schema.json` and `docs/scan.schema.json`; the risk classes `safe`, `rebuildable`, `review` and `user-data`; live versus dead.
- **What to do with it.**
  - Tell the person what is there and what is safe.
  - Give them the exact `ddc clean` command or the app's Review window to do it themselves.
- **What never to do.**
  - Run `ddc clean` with `--yes`, or with `--items` it wrote itself.
  - Delete a finding with `rm -rf`.
  - Remove a worktree with `git worktree remove --force`.
  - Touch a live session's files.
  
  This follows the ADR "Agent footprint: worktrees, harness data and bounded consent for agents", §5 (an agent's list carries no person's consent).

Deliverables in the code repository:
- `agents/claude-code/ddc/SKILL.md` (Claude Code skill);
- `agents/AGENTS.snippet.md` (Codex, opencode and any harness that reads `AGENTS.md`), with begin and end markers;
- `agents/README.md`: how to install them, for a person or by an agent that is asked to.

The full story later replaces v0 with `footprint` and `clean --agent`.

## Decomposition

- [ ] The three files. Every rule they state cites the ADR section or schema field it comes from.
- [ ] A structure test in `crates/ddc-cli/tests/structure.rs`: every schema field the skill names exists in the schema it cites, and every `ddc` command line in the skill parses (`--help` of that subcommand succeeds).
- [ ] README: a "For coding agents" section linking `agents/`.
- [ ] A try-out in a Claude Code session with the skill installed, in a scratch project on a synthetic home: asked "why is my disk full", it runs `ddc disk agents --json` and `ddc scan`, and proposes rather than cleans. Record the commands it ran, not the transcript.

## Implementation Plan

## Acceptance Criteria

- [ ] The three files exist; the structure test passes in the gate (CI).
- [ ] The skill contains no command that removes files, other than in its "what never to do" list and as text for the person to run (a test greps the runnable code blocks).
- [ ] (After the deadline; it does not block v0) The try-out shows the agent running only read-only `ddc` commands and handing the cleanup to the person (recorded command list).

## Final Summary

## Technical Notes

- `--json` output always goes to stdout, and the terminal warning only to a terminal's stderr. An agent's captured output is pure JSON (README, "The command line").

## Dependencies

- Story "Per-session footprint: what each agent session keeps in its transcripts, temporary folders and worktrees", for the per-session fields. Until it is released, the skill describes them as "when available" and works with the current schema.

## Attachments

-

---

*Last updated: 2026-10-31*

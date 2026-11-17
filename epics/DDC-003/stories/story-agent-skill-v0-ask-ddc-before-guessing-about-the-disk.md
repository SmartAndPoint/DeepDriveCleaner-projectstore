---
type: story
id: "story-agent-skill-v0-ask-ddc-before-guessing-about-the-disk"
epic: "DDC-003"
title: "Agent skill v0: ask ddc before guessing about the disk"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-31
updated: 2026-11-17
external_refs: {}
tags: [agents, skill, agents-md, docs]
code_refs: ["agents/claude-code/ddc/SKILL.md", "agents/AGENTS.snippet.md", "agents/README.md", "crates/ddc-cli/tests/structure.rs"]
specs: []
started_at: "2026-11-01T00:00:00.000Z"
closed_at: "2026-11-17T16:11:04.793Z"
plan_updated_at: "2026-11-17T15:57:24.091Z"
status_changed_at: "2026-11-17T16:11:04.793Z"
---

# Agent skill v0: ask ddc before guessing about the disk

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | done |
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

- [x] The three files. Every rule they state cites the ADR section or schema field it comes from. — done (PR #58): `agents/claude-code/ddc/SKILL.md`, `agents/AGENTS.snippet.md`, `agents/README.md`
- [x] A structure test in `crates/ddc-cli/tests/structure.rs`: every schema field the skill names exists in the schema it cites, and every `ddc` command line in the skill parses (`--help` of that subcommand succeeds). — done (PR #58): three tests in `crates/ddc-cli/tests/structure.rs`, schema paths resolved through `$defs`
- [x] README: a "For coding agents" section linking `agents/`. — done (PR #58)
- [x] A try-out in a Claude Code session with the skill installed, in a scratch project on a synthetic home: asked "why is my disk full", it runs `ddc disk agents --json` and `ddc scan`, and proposes rather than cleans. Record the commands it ran, not the transcript. — moved to runbook "Owner checks: manual acceptance across stories", section "The agent skill v0 in a live session"
- [x] `ddc changes --json` as the first line of "What to run" in `agents/claude-code/ddc/SKILL.md` and `agents/AGENTS.snippet.md`, before `ddc scan "$PWD"`, commented "what grew between the person's last two scans, their own cleans apart; run it before a scan, which replaces the latest report", and one sentence that `status: not-compared` with its `reason` is the normal answer until two whole scans are saved. The bound of ADR "Agent footprint: worktrees, harness data and bounded consent for agents" §7 in plain words: showing what grew is fine, removing it is not the agent's; it lists the whole home, so show the person only what they asked about. `agents/README.md` names it beside the other read-only commands. `every_ddc_line_of_the_agent_guidance_parses` adds `changes` to the questions the skill must ask; the line cites no field of `docs/changes.schema.json`, or `the_agent_guidance_names_only_schema_fields` gains a list of that schema's names. Moved here from the story "ddc changes: what grew between the last two scans" (DDC-001; the command shipped in 0.34.0), since this story is open. — done (PR #170)

## Notes

- 0.14.1 (PR #59): the skill cites the released footprint fields (`sessions.footprints[]`, `sessions.temp_root`, the row fields and `refused`) and the snippet names `sessions.footprints[]`, `state` and `sessions.temp_root`, checked by the structure test; no name is left as "when available".

## Implementation Plan

Written 2026-11-17, after 0.37.0. No spec: the work is guidance text and one structure test (`no_spec`). What is left is the open Decomposition item on `ddc changes`, one sentence that 0.37.0 made untrue, and the try-out.

1. `agents/claude-code/ddc/SKILL.md`, "What to run": `ddc changes --json` right after `ddc --version` (that line checks the install; it asks the disk nothing) and before `ddc agent status`, with a short comment. The prose under the block says why it comes first: `ddc scan "$PWD"` saves its report as the latest and moves the person's whole scan to `previous-scan.json` (`store::write_last_scan`), so after it `ddc changes` compares that project only and the person's last two whole scans are no longer the pair. The agent never runs a whole scan to get a comparison.
2. A new section, "Reading `ddc changes --json` (docs/changes.schema.json)", before the one on `ddc agent status`: `status` is `compared` or `not-compared`; with `not-compared`, `reason` says why, and `no-report` or `no-previous` is the normal answer until two whole scans are saved. The bound of ADR "Agent footprint: worktrees, harness data and bounded consent for agents" §7 (its 2026-11-04 note) in plain words: showing what grew is fine, the builds in coding agents' worktrees included; removing it is not the agent's; the report covers the whole home, so show the person only what they asked about.
3. The last sentence of SKILL.md, "A bounded cleanup for agents is planned (the same ADR, §5); until it ships, cleaning is the person's", is untrue since 0.37.0. It now says that `ddc agent clean` exists and that this guidance does not use it: cleaning stays the person's here. Guidance on `ddc agent footprint` and `ddc agent clean` is the story "Agent guidance: skill and AGENTS.md snippet" (DDC-003), as is the skill's `allowed-tools`.
4. `agents/AGENTS.snippet.md`: the same line first in its block, and one sentence in "How to read it". `agents/README.md`: the command beside the other read-only ones, and `changes.schema.json` beside the cited schemas. The top-level README's "For coding agents" paragraph names it among the commands.
5. `crates/ddc-cli/tests/structure.rs`: `CHANGES_NAMES` (`status` with `not-compared` and `compared`, `reason` with `no-report` and `no-previous`), checked against `docs/changes.schema.json` like the other two lists; `every_ddc_line_of_the_agent_guidance_parses` asks for `changes` too. Pitfalls: a name in backticks must be one word (`status: not-compared` in one pair of backticks is skipped by the name filter, unchecked), and every name and value of a list must appear in SKILL.md. No text in `agents/` may say `settings set` (the settings switch's test).
6. The try-out moves to the runbook "Owner checks: manual acceptance across stories" at close: it needs a Claude Code session with the skill installed on a synthetic home, which is the owner's.

One code PR, built and tested on the Windows host's WSL and the native Windows leg, then the close.

## Acceptance Criteria

- [x] The three files exist; the structure test passes in the gate (CI). — evidence: PR #58's required checks green, main's Gate run on the merge; `cargo test -p ddc-cli --test structure` (18 passed); reviewer
- [x] The skill contains no command that removes files, other than in its "what never to do" list and as text for the person to run (a test greps the runnable code blocks). — evidence: `the_agent_guidance_runs_nothing_that_removes` (no `ddc clean`, `disk free --actions`, `rm`, `mv`, `--yes`, `--items` in a runnable block), PR #58, reviewer
- [x] The guidance runs `ddc changes --json` first, read-only, with the §7 bound and the "show only what was asked" rule. — evidence: `every_ddc_line_of_the_agent_guidance_parses` asks for `changes`; `the_agent_guidance_names_only_schema_fields` checks `CHANGES_NAMES` against `docs/changes.schema.json`; PR #170's required checks and Gate green; reviewer
- [x] The guidance no longer says the bounded cleanup for agents is still to come; it names `ddc agent clean` as one it does not use. — evidence: `agents/claude-code/ddc/SKILL.md`, the last paragraph of "What never to do" (PR #170); reviewer
- [x] (After the deadline; it does not block v0) The try-out shows the agent running only read-only `ddc` commands and handing the cleanup to the person (recorded command list). — moved to runbook "Owner checks: manual acceptance across stories", section "The agent skill v0 in a live session"

## Final Summary

**What changed.** PRs #58 and #59 shipped the read-only guidance in `agents/` (a Claude Code skill, an `AGENTS.md` snippet, their README) and three structure tests. PR #170 finished it: the skill and the snippet run `ddc changes --json` first, right after `ddc --version`, and say how to read `status`, `reason` and `compared_roots`; the person sees only what they asked about, and removing what grew stays theirs. The skill's last sentence no longer says a bounded cleanup for agents is to come: `ddc agent clean` shipped in 0.37.0, and this guidance does not use it. `agents/README.md` and the README name the command and its schema.

**Why.** A project scan replaces the latest report, so after `ddc scan "$PWD"` the person's comparison of two whole scans is gone and `ddc changes` compares that project only (`store::write_last_scan`). Asking first keeps the answer the person would get.

**Deviations, decided while building.** (1) The line follows `ddc --version`, which checks the install and asks the disk nothing. (2) The reviewer found "it covers the whole home" true only for two scans of the home: the text names `compared_roots` instead, and says `no-report` or `no-previous` is the normal answer before two scans are saved. (3) "Removing it is not yours" cites this guidance, not §7, since §7 lets an agent clean `safe` items behind the opt-in. (4) Guidance on `ddc agent footprint` and `ddc agent clean`, and the skill's `allowed-tools`, stay with the story "Agent guidance: skill and AGENTS.md snippet".

**Tests.** `crates/ddc-cli/tests/structure.rs`: `CHANGES_NAMES` checked against `docs/changes.schema.json`, `changes` among the questions the skill must ask. On a Linux machine, clippy and the structure tests (33 passed) and the pre-push gate-fast (1132 passed); the native Windows leg (1153 passed, the skips as listed); PR #170's Gate.

**Risks and follow-ups.** The try-out in a live Claude Code session is the owner's check in the runbook. Whether the skill triggers reliably is open until it has been used.

## Status log

- 2026-11-17T16:11:04.793Z status: in-progress → done — closed (close gate)

## Technical Notes

- `--json` output always goes to stdout, and the terminal warning only to a terminal's stderr. An agent's captured output is pure JSON (README, "The command line").

## Dependencies

- Story "Per-session footprint: what each agent session keeps in its transcripts, temporary folders and worktrees", for the per-session fields. Until it is released, the skill describes them as "when available" and works with the current schema.

## Attachments

-

---

*Last updated: 2026-11-17*

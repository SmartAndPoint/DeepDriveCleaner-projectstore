---
type: story
id: "story-agent-guidance-skill-and-agents-snippet"
epic: "DDC-003"
title: "Agent guidance: skill and AGENTS.md snippet"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-11-18
external_refs: {}
tags: [agents, docs, skill]
code_refs: ["agents/", "crates/ddc-core/src/guidance/", "crates/ddc-cli/src/cmd/agent_skill.rs", "docs/skill.schema.json", "crates/ddc-cli/tests/structure.rs"]
specs: ["agent-skill-install", "settings-file-writers"]
started_at: "2026-11-17T16:51:13.125Z"
closed_at: "2026-11-18T06:14:14.091Z"
plan_updated_at: "2026-11-17T16:51:13.125Z"
status_changed_at: "2026-11-18T06:14:14.091Z"
---

# Agent guidance: skill and AGENTS.md snippet

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | done |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

*Amended 2026-11-02 (ADR "Agent footprint", §7): `ddc agent skill install` / `status` install and report the skill and the snippet; the guidance names `ddc agent status`, `ddc agent footprint` and `ddc agent clean`, and proposes installing `ddc` only after the person agrees. Amended 2026-11-17 (plan gate): the Claude Code marketplace plugin is a later story (the owner).*

The code repository gets an `agents/` folder with what a coding agent needs to use `ddc` well: §6 of the ADR "Agent footprint: worktrees, harness data and bounded consent for agents". A Claude Code skill (`agents/claude-code/ddc/SKILL.md`) and an `AGENTS.md` snippet (`agents/AGENTS.snippet.md`, for Codex, opencode and any harness that reads `AGENTS.md`) say when to run `ddc agent footprint`, what `ddc agent clean` may and may not do, how to read the two JSON contracts, and that a worktree is removed through `ddc`, never through the agent's own `git worktree remove`. The guidance decides nothing: every rule it states cites the ADR section or the schema field it comes from.

The MCP server is deferred by the ADR; this story does not start it.

## Decomposition

Rewritten at the plan gate, 2026-11-17: the names follow ADR "Agent footprint" §7 (`ddc agent footprint`, `ddc agent clean`, `ddc agent skill install|status`). What 0.37.0 shipped is the base: `ddc agent status`, `ddc agent footprint`, `ddc agent clean` without `--session`, and the person's switch. The guidance v0 is in `agents/` (story "Agent skill v0: ask ddc before guessing about the disk").

- [x] PR 0, vault: this plan, the spec "Agent skill install" and the ADR's dated note (vault #146)
- [x] PR 1, guidance: `SKILL.md`, the snippet and both READMEs name `ddc agent footprint` and `ddc agent clean`, with `allowed-tools` and `metadata.version`; the structure tests re-premised (#171)
- [x] PR 2, core: the embedded guidance, the marker and the `AGENTS.md` block in `crates/ddc-core/src/guidance/`; `.claude/skills` sealed (#173)
- [x] PR 3, CLI: `ddc agent skill install|status`, `docs/skill.schema.json`, `make bump` writing `metadata.version`, the typo filter per command; the guidance names the install (#172 (the `--from-app` rename and the typo filter) and #175)
- [x] The live try-outs move to the runbook "Owner checks: manual acceptance across stories" at close — done: section "Agent guidance and ddc agent skill install"

## Implementation Plan

Written 2026-11-17 after the planner's pass. Covered by the spec [[specs/agent-skill-install|Agent skill install]] (`skill-install`): PR 2 and PR 3 are a route through its contracts; PR 1 is guidance text under this story's own criteria.

**Decisions.**

- D1. The Claude Code marketplace plugin is a later story (the owner, 2026-11-17): no SmartAndPoint marketplace exists, and a plugin would move `SKILL.md`, add a version to `make bump` and put a manifest outside the MIT paths. `status` does not read Claude Code's plugin data.
- D2. `metadata.version` in `SKILL.md` is the release's version. `make bump` writes it and a test checks it, as the ADR says. After each release, `status` calls an installed copy `older` until it is installed again.
- D3. The marker's hash is FNV-1a, 64 bits, written in place: it finds edits and is not a security boundary, so it adds no dependency.
- D4. `allowed-tools` names only commands that read: `ddc agent status`, `ddc agent footprint`, `ddc agent clean --dry-run`, `ddc changes` and `ddc rules`, each under `deepdrivecleaner` too: ten entries, in the pattern form Claude Code's skill documentation gives, checked against PR 1's exact list. `--dry-run` follows `clean` at once, since the patterns match by prefix; Claude Code splits a command at `;`, `&&`, `|` and the like before it matches, so a pattern never covers a second command. `ddc scan` stays a prompt, because it replaces the person's latest report; so do `ddc agent clean` and `skill install`.
- D5. On Windows both new commands are refused like every command except `rules` (spec contract 12).
- D6. `ddc agent clean --session` does not ship yet (story "ddc agent clean --session: clean up after one session"): the guidance does not name it, and that story adds it.
- D7. PR 1 cannot name `skill install`, since every `ddc` line of the guidance must parse; PR 3 adds it.
- D8. The app's hidden flag of `ddc settings set` becomes `--from-app` (it was `--approved`), and the typo filter works per command. Otherwise `skill install --help` would name, to every agent, the flag that switches the opt-in on without a terminal (critic, 2026-11-17; spec "Settings file writers", contract 10, amended). The app passes the new name in the same PR; it ships its own `ddc`, so the two never disagree.

**PR 1, the guidance** (`agents/claude-code/ddc/SKILL.md`, `agents/AGENTS.snippet.md`, `agents/README.md`, `README.md`, `crates/ddc-cli/tests/structure.rs`).

- When: at the start of a session in a worktree, before installing dependencies into a new one, on `ENOSPC` or a failed install, at the end of a session (§6). What: `ddc changes --json` before any scan; `ddc agent footprint --json` for the project; `ddc agent clean --dry-run --json`, then `ddc agent clean --json`, inside the project only (§7).
- How to read: the footprint's checkouts and totals (`safe` apart from `rebuildable`); the clean's skips (`agent-opt-in-off`: the person can allow it in the app's Settings; `not-consented`: the person's), and the `ddc clean --items -` command it prints for the person, shown in a `text` block and never run. A worktree goes through `ddc`, never the agent's own `git worktree remove`.
- No `ddc` installed: propose the Homebrew cask or the install script, and run it only after the person says yes (§7).
- Tests: field lists for `docs/footprint.schema.json` and `docs/clean.schema.json` beside the three there are; `agent footprint` and `agent clean` among the questions; `the_agent_guidance_runs_nothing_that_removes` allows `ddc agent clean` in a runnable block only without `--items`, and still bans `ddc clean`, `--items`, `--yes` and the rest; a test reads `allowed-tools` (every entry parses, none is `agent clean` without `--dry-run` or `skill install`). No text in `agents/` says the settings command (its guard test).

**PR 2, core** (`crates/ddc-core/src/guidance/`, `crates/ddc-core/src/rules/catalog/shared/agents.rs`, `crates/ddc-core/tests/guidance.rs`, `crates/ddc-core/tests/agent_rules.rs`). Contracts 2 to 7 and the states of 9 and 10; `.claude/skills` sealed in the home's Claude Code rules (ADR §7), with a rule test that a scan of a home with `~/.claude/skills/ddc/` finds nothing under it. A project's `.claude/skills` is protected by ADR §2: untracked, it makes a worktree dirty, and ignored, it is ignored content of the worktree's own; either way the worktree is held back. The guidance files are built in with `include_str!`, outside `ddc-core/src/agents/`, the read-only reader whose test keeps it from cleanup. `store::write_atomic` writes; the code checks the skill folder is no link before it, since `create_dir_all` follows links.

**PR 3, CLI** (`crates/ddc-cli/src/cli.rs`, `cmd/agent_skill.rs`, `cmd/mod.rs`, `gate.rs`, `tests/agent_skill_cli.rs`, `tests/cases.json`, `docs/skill.schema.json`, `scripts/bump-version.sh`, and for D8 `cmd/settings_set.rs`, `tests/settings_set_cli.rs` and DDCKit's `AgentsOptIn.swift` with its tests). Contracts 1, 5 (the bump), 8, 9 to 11 and 12, D8, with `cmd/settings_set.rs`'s pattern for the root refusal, the terminal check and the question. `without_hidden_hints` works per command, so the visible `--approved` of `skill install` cannot bring back the hint `settings set` hides. The guidance and both READMEs then name `ddc agent skill install --approved` in a `text` block, for after the person's yes.

Each PR is built and tested on the Windows host's WSL and the native Windows leg; new Windows skips are listed.

## Acceptance Criteria

- [x] Test (`structure.rs`): the skill's frontmatter has `name`, `description`, `allowed-tools` and `metadata.version`, and every `ddc` command line in both files parses, `agent footprint` and `agent clean` among the questions the skill asks — evidence: `structure.rs` `the_agent_guidance_runs_nothing_that_removes` (name, description triggers), `every_ddc_line_of_the_agent_guidance_parses` (asks `changes`, `agent footprint`, `agent status`, `scan`, `rules`, `agent clean`), `the_skill_allows_only_what_reads_and_carries_the_version` (PR #171)
- [x] Test: every JSON field the two files name exists in the schema they cite, `docs/footprint.schema.json` and `docs/clean.schema.json` among them — evidence: `the_agent_guidance_names_only_schema_fields` with `FOOTPRINT_NAMES` and `CLEAN_NAMES` checked against `docs/footprint.schema.json` and `docs/clean.schema.json` (PR #171)
- [x] Test: a runnable block runs no removal but `ddc agent clean` without `--items`; `allowed-tools` is exactly D4's ten entries — evidence: `the_agent_guidance_runs_nothing_that_removes`, `the_skill_allows_only_what_reads_and_carries_the_version` (`assert_eq!` with `ALLOWED_TOOLS`, both names) (PR #171)
- [x] Test (`agent_rules.rs`): a scan of a home with `~/.claude/skills/ddc/` has no finding at or under `~/.claude/skills` — evidence: `agent_rules.rs` `sealed_folders_and_auto_memory_are_never_findings` with `.claude/skills` and a project under it (PR #173)
- [x] Test (`settings_set_cli.rs`): `settings set` refuses `--approved` and never suggests `--from-app`; the app passes `--from-app` (DDCKit `AgentsOptInTests`) — evidence: `settings_set_cli.rs` `a_typo_is_never_answered_with_the_hidden_option` (`--from-ap`, `--fromapp`, `--fron-app`, `--app`, `--approved`; nothing written; `--data-dir` hidden on settings set, shown on disk docker), DDCKit `AgentsOptInTests` `toggleRunsTheCLI` with `--from-app`, `make ddckit-test` 608 tests (PR #172)
- [x] Review: neither file says what ADR "Agent footprint: worktrees, harness data and bounded consent for agents" does not (§4 to §7), checked by the reviewer and listed in the final summary — evidence: four reviewer passes (PRs #171, #172/#173, #175), deviations in the Final Summary
- [x] The acceptance of the spec "Agent skill install" holds — evidence: the spec's acceptance, checked in the same vault PR
- [x] Every PR's Gate is green, Linux and native Windows legs included — evidence: PRs #171, #172, #173, #175 merged with required checks green; native Windows leg 1160, 1178 and 1183 tests passed with the skips as listed; GitHub's hosted macOS runners dropped jobs several times and they were run again
- [x] Owner check (runbook "Owner checks: manual acceptance across stories"): a Claude Code session with the skill installed, in a fresh worktree and asked to install the project's dependencies, runs `ddc agent footprint` first; a second one asked to "free some space in this project" runs `ddc agent clean` and nothing else that removes files — moved to runbook "Owner checks: manual acceptance across stories", section "Agent guidance and ddc agent skill install"

## Final Summary

**What changed.** Four code PRs on the plan's route. #171: the guidance in `agents/` names `ddc agent footprint` and `ddc agent clean` (when to ask, how to read both reports, the person's `ddc clean --items -` command shown and never run, a worktree never the agent's), proposes installing ddc only after a yes, and the skill's frontmatter carries `allowed-tools` (ten entries that only read) and `metadata.version`, which `make bump` now writes. #173: `crates/ddc-core/src/guidance/` builds the skill and the `AGENTS.md` block into the binary, with the `.ddc-skill` marker (FNV-1a per file), the folder's and the file's states, plans before writes, and `docs/skill.schema.json`; `.claude/skills` is sealed. #172: the app's hidden flag of `settings set` became `--from-app`, and the typo filter works per command. #175: `ddc agent skill install|status` on that core, with consent at a terminal or `--approved`, cases, the gate, and the guidance naming the install.

**Why.** ADR "Agent footprint" §6 and §7: the agent's guidance should name the commands 0.37.0 shipped, and installing it should be one command that writes ddc's own text and removes nothing.

**Deviations, decided while building.** (1) D1: the Claude Code plugin is a later story (the owner, 2026-11-17). (2) The reviewers found the guidance stricter than §7 in places; it now runs `ddc agent clean` "when you clean up after yourself: at the end of a session, when the person asks, or when a lack of space stops your work", and forbids passing `--items` to it, stricter than the ADR, which allows a list as a filter. (3) A shipped file the marker does not list, present and different, counts as edited (spec contract 3, amended), so a future file never overwrites a person's own. (4) The embedded-copies test is in `crates/ddc-core/tests/guidance.rs`, and contract 11 is held by the structure guard, which bans locks, journals, removals and commands in the installer, not by a CLI lock test (spec Testing, updated). (5) `--agents-md` also writes the home's Claude Code skill; the snippet says so. (6) A cargo deny failure on main (chrono and humantime unmaintained) was fixed by PR #174 at the owner's choice, outside this story.

**Tests.** `structure.rs` guidance tests re-premised and two new (allowed-tools and version; the installer guard); `scripts/tests/test_bump_version.py`; `crates/ddc-core/tests/guidance.rs` (16 tests); `agent_rules.rs`; `crates/ddc-cli/tests/agent_skill_cli.rs` (6 tests, a pseudo-terminal run among them); `settings_set_cli.rs`; six cases; the cases and README checks now handle three-word commands. Each PR: clippy and the unit tier on a Linux machine, the native Windows leg, the PR's Gate.

**Risks and follow-ups.** The live try-outs are the owner's (runbook). Whether the skill triggers reliably is open until it has been used. The per-harness stories (the Claude Code plugin, `~/.agents/skills` for Codex, opencode, Gemini CLI and Qwen Code, Antigravity, hooks) and the move from humantime to jiff follow in the backlog.

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Status log

- 2026-11-17T16:51:13.125Z status: planned → in-progress — work started (plan gate)
- 2026-11-18T06:14:14.091Z status: in-progress → done — closed (close gate)

## Technical Notes

- Claude Code loads skills from `.claude/skills/<name>/SKILL.md` in the project and `~/.claude/skills/` for the user; a skill's `description` is what decides whether it is used. The frontmatter is the only machine-read part; the body is instructions.
- The `AGENTS.md` snippet's markers are `<!-- ddc:begin -->` and `<!-- ddc:end -->` (v0, PR #58); `ddc agent skill install --agents-md` replaces the block between them (spec "Agent skill install", contracts 6 and 7). *Amended 2026-11-17: the earlier `ddc:agents` style was never shipped.*
- A skill that says "run `ddc agent clean`" is only as safe as that command's bound and the person's opt-in; the guidance adds no permission, and the test that commands parse keeps it from drifting from the CLI.
- The "try it" check is one transcript read, not an evaluation suite; whether the skill triggers reliably is a later question, once the skill has a few weeks of use.

## Dependencies

- [[epics/DDC-003/stories/story-ddc-footprint-scoped-project-report|ddc footprint: a scoped project report]]
- [[epics/DDC-003/stories/story-clean-agent-bounded-consent|Clean: --agent bounded consent]]
- [[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]]

## Attachments

- Claude Code skills: https://code.claude.com/docs/en/skills

---

*Last updated: 2026-11-18*

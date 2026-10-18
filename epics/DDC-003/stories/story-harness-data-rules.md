---
type: story
id: "story-harness-data-rules"
epic: "DDC-003"
title: "Harness data rules"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-18
external_refs: {}
tags: [rules, agents, safety]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Harness data rules

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

What Claude Code, Codex and opencode keep under the home folder gets rules: §3 of the ADR "Agent footprint: worktrees, harness data and bounded consent for agents". Scratch directories of Claude Code are `safe`; `file-history` and the Codex and opencode data are `review`; transcripts are reported, not removed; Claude Code's auto memory is never inside a finding. Each rule gets `origin` with `kind: session-data` or `runtime-cache`.

The paths come from the harnesses' documentation and from the research note; the plan gate confirms each on the reference machine before it enters the catalog, and a path that does not exist there enters only with a fixture.

## Decomposition

Rough until the plan gate.

- [ ] Known-path rules in `rules/catalog.rs`, with `origin`:
  - `agent.claude-code-scratch`: `.claude/paste-cache`, `.claude/debug`, `.claude/shell-snapshots`, `.claude/usage-data`, `.claude/todos`, `.claude/statsig`, `.claude/logs` — `safe`, `delete`, owner: the `claude` process (matched by executable name, since it is not an app bundle)
  - `agent.claude-code-file-history`: `.claude/file-history` — `review`, `trash`
  - `agent.claude-code-transcripts`: `.claude/projects` — `review`, `none`, a **region**: walked for size, nothing inside claimed, except that `projects/<project>/memory/` at any depth is excluded from the finding's bytes and marked kept in the report
  - `agent.codex-sessions`: `.codex/sessions`, `.codex/log` — `review`, `trash`
  - `agent.codex-runtimes`: `.cache/codex-runtimes` — `review`, `trash`
  - `agent.opencode-data`: `.local/share/opencode`, `.cache/opencode` — `review`, `trash`
- [ ] The hidden-directory rule of the catalog already downgrades project artifacts under `~/.claude`, `~/.codex`, `~/.local` to `review`; a `node_modules` inside `~/.claude/plugins` therefore stays `review` as today, and `~/.claude/plugins` itself gets no rule (installed plugins are not residue)
- [ ] Hints: transcripts → "Claude Code removes these after cleanupPeriodDays (30 by default); `claude purge` removes one project's"; `file-history` → "checkpoint restore for past sessions"; Codex and opencode → "session history of the harness"
- [ ] `ddc rules --json` (DDC-002) lists the new rules under a group "Coding agents"; no dependency, the group label is data in the catalog
- [ ] Plan gate: on the reference machine, `ls -la ~/.claude ~/.codex ~/.local/share/opencode ~/.cache/codex-runtimes ~/.cache/opencode` and `du -sk` on each; decide which paths exist, which are a region, and whether the Codex session layout maps to a project (feeds the epic's open question)
- [ ] `bench/make_baseline.py`: expected rule and class for the new paths, so the coverage check of "Detection rule catalog" covers them

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test (`rules.rs`, fixtures under a fake home): each path of the six rules is found with its id, class, method and `origin`; `every_rule_is_covered_by_a_fixture` stays green
- [ ] Test: `~/.claude/projects/<p>/memory/MEMORY.md` and `~/.claude/projects/<p>/memory/notes/x.md` lie inside no finding, the `agent.claude-code-transcripts` finding's `bytes` exclude them, and a `.venv` with `pyvenv.cfg` inside `~/.claude/projects/<p>/<session>/tool-results/` is **not** a `python.venv` finding (the region claims nothing but counts everything)
- [ ] Test: `agent.claude-code-transcripts` with action `delete` or `trash` is `action-not-allowed` in the executor (`catalog_invariants` extended: every rule with `origin.kind = session-data` and method `none` has class `review` or `user-data`)
- [ ] Test (`clean_safety.rs`): `agent.claude-code-scratch` is skipped `owner-running` while a fake process named `claude` runs, and removed when none does
- [ ] Test: `node_modules` with `package.json` under `~/.claude/plugins/x/` is `review` (hidden-directory rule) and `~/.claude/plugins` itself is not a finding
- [ ] On the reference machine, after the plan gate: `ddc scan ~` lists each confirmed path with the expected rule, `python3 -I bench/check_baseline.py` covers the new rows, and the final summary records the sizes found and which documented paths did not exist there
- [ ] `make check` passes

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- Claude Code's own retention sweep deletes transcripts, `file-history`, `paste-cache`, `debug`, `usage-data` and the legacy `todos`/`statsig`/`logs` once older than `cleanupPeriodDays` (default 30). `shell-snapshots` are removed on clean exit and swept after a crash. `projects/<project>/memory/` is never swept. So `ddc`'s `safe` rules only shorten that window; the `review` rule for `file-history` and the report-only rule for transcripts never race it.
- Newer Claude Code versions keep pasted images and the session scratchpad under `CLAUDE_CODE_TMPDIR` (the system temp directory), outside `$HOME`; `ddc` does not look there.
- A region finding that excludes a subtree from its bytes is new: `~/Downloads` is a region where rules keep firing, `~/.claude/projects` is a region where nothing fires and one subtree is kept. The visitor needs a `Region` tag variant that records kept subtrees, or `memory/` is treated like `.ddc-keep` inside the region; the plan gate decides.
- Owner matching by executable name (`claude`) differs from the bundle matching of "Safe cleanup: ddc clean" (`OwnerApp`); the `ProcessTable` already exposes executable paths, so a name-based owner is a small addition, kept separate from the bundle path so the `CursorUIViewService` lesson is not repeated: it matches `argv[0]`'s file name exactly, never a substring.
- Codex and opencode layouts are from their documentation and the research note; they are the least certain part of this story and the reason for the plan-gate check.

## Dependencies

- [[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]]
- [[epics/DDC-001/stories/story-detection-rule-catalog|Detection rule catalog]]

## Attachments

- Claude Code application data: https://code.claude.com/docs/en/claude-directory

---

*Last updated: 2026-10-18*

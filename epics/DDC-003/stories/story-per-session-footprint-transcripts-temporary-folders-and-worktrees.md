---
type: story
id: "story-per-session-footprint-transcripts-temporary-folders-and-worktrees"
epic: "DDC-003"
title: "Per-session footprint: what each agent session keeps in its transcripts, temporary folders and worktrees"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-31
updated: 2026-11-01
external_refs: {}
tags: [agents, disk-agents, sessions, worktrees, temp, json]
code_refs: ["crates/ddc-core/src/agents/footprint.rs", "crates/ddc-core/src/agents/sizes.rs", "crates/ddc-core/src/agents/model.rs", "crates/ddc-core/src/agents/mod.rs", "crates/ddc-core/src/env.rs", "crates/ddc-cli/src/cmd/disk_agents.rs", "crates/ddc-cli/src/cli.rs", "crates/ddc-cli/src/render.rs", "docs/agents.schema.json", "crates/ddc-core/src/platform/macos/meta.rs", "crates/ddc-core/src/platform/linux/meta.rs", "crates/ddc-core/tests/agents.rs", "crates/ddc-cli/tests/disk_agents_cli.rs"]
specs: []
started_at: "2026-11-01T00:00:00.000Z"
closed_at: null
plan_updated_at: "2026-11-01T00:00:00.000Z"
---

# Per-session footprint: what each agent session keeps in its transcripts, temporary folders and worktrees

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | in-progress |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner's priority (2026-10-31): "working with the disk — show how much each agent session ate in its own working files, and how to clean it; and a skill for agents". `ddc disk agents` today shows each harness's folders, its live sessions, and a count of other transcripts. It does not show:
- how much a single session costs;
- that session's working files outside the home folder;
- its worktrees.

On the reference Mac, a Claude Code session's temporary folder held up to 9 GB. The whole `/private/tmp/claude-<uid>` held 15 GB, more than the 12 GiB the report found under the home folder.

This story adds the per-session view. Each session or thread gets one row:
- harness, project, session id, live or dead, last activity;
- the bytes it keeps, by kind:
  - **transcript**: Claude Code's `~/.claude/projects/<project>/<session>.jsonl` and its `<session>/` companion; Codex's `rollout-*.jsonl`;
  - **checkpoints**: Claude Code's `~/.claude/file-history/<session>`;
  - **temporary folder**: Claude Code's `<tmp>/claude-<uid>/<project-slug>/<session>/`, which holds `scratchpad`, `tasks` and `images`;
  - **worktrees**: git worktrees attributable to the session. A worktree counts when its path is inside the session's temporary folder, or when the research step finds a reliable link between a harness's own worktree folder (Codex's `~/.codex/worktrees/<id>`, Claude Code's worktree folder if it has one) and a session;
- the total;
- for a dead session, how to clean it. Until the cleanup rule exists, this is a description, not a command: "dead session, 3.4 GB in its temporary folder (`<path>`), 1 worktree of `<repo>`". The rule is story "Coding agents' temporary folders outside the home", amended into this story's cleanup half below.

Rows are sorted by total, largest first. The text output shows the top 10 per harness and "N more, X GB". `--json` lists them all.

**Scope for the challenge deadline (2026-10-31 23:59 PDT): an MVP.**
- Claude Code only: transcript, checkpoints and temporary-folder bytes, live or stale, as additive JSON plus the text top 10.
- Worktree attribution and the Codex footprint are a follow-up in this story, after the research step.
- The Mac app's rows and the cleanup half come after.

This reverses one sentence of the agent ADR §3 for reading: "the OS clears them, and `ddc` does not look for them". A dated amendment says `disk agents` now looks at the current user's folder and reports it, read-only. Removal stays out; that is story "Coding agents' temporary folders outside the home".

**Split between the two stories.** The report-side code (the seam, the schema, the text) lives here. That story owns the research, the removal amendments, the rule and the app row.

**What it reads.** Names, sizes, the process list and the session records' named fields, as `disk agents` does today. For worktrees (follow-up), it reads a worktree's `.git` file (`gitdir:`), as `report/project.rs` already parses it, never by running git in a worktree. It never reads a transcript's content.

`crates/ddc-cli/tests/structure.rs` forbids the substrings `trash`, `remove_` and `"delete"` in `ddc-core/src/agents/*` and `disk_agents.rs`. The "how to clean" wording must avoid them there, or come from elsewhere.

## Decomposition

- [ ] Research: confirm the temporary-folder layout and its location on macOS and Linux (`/tmp` versus `$TMPDIR`; `/tmp` is a symlink to `/private/tmp` on macOS), the `<project-slug>` rule, and whether the folder name equals the transcript's session id. Also how Codex's `~/.codex/worktrees/<id>` maps to a thread, and whether Claude Code keeps worktrees elsewhere. Add it to the research note "Coding-agent harnesses on macOS: where their data lives and how to tell a live session", with the critic.
- [x] The ADR amendment for reading (agent ADR §3), with the critic. — done: the amendment of 2026-11-01, vault PR #36
- [x] `agents/` model: `SessionFootprint { id, project, state, live_by, unknown_cause, modified, transcript, checkpoints, temp, temp_path, total }`, where `state` is a new enum (`live | not-live | unknown`, from `SessionLiveness`), not a new variant of `SessionState`, which would bump the schema version; `worktrees: [{path, repo, bytes}]` joins it in the follow-up. The temporary root comes through the existing `Env::system_root()` ("a fixture root in tests"), so tests never touch the real `/tmp`. — done (PR #58): `crates/ddc-core/src/agents/model.rs`; worktrees follow
- [x] Sizing reuses `agents/sizes.rs` and the parallel walker; `--no-sizes` still skips it. A worktree inside a temporary folder is counted once: in `worktrees`, and excluded from `temp`. — first sentence done (PR #58): `agents/sizes.rs`, `agents/footprint.rs`; the worktree half follows with worktree attribution
- [x] `docs/agents.schema.json`: `sessions.footprints[]`, an optional, additive list beside the session list, and `sessions.temp_root` (the temporary base's path and bytes, so what no session accounts for, such as `bash-edit-diff`, shows as the remainder), with **no** `schema_version` bump. A sibling list, not a field of each session: the session list holds only sessions with a record, and Claude Code removes a record when its session ends, so most finished sessions would get no row. DDCKit's reader rejects any other version (`AgentsReport.swift`). The existing fixture still decodes (contract test). A bump, if ever wanted, is its own PR with both constants. — done (PR #58): `docs/agents.schema.json`, version 1
- [x] Text output: the per-session table, top 10 per harness. — done (PR #58): `crates/ddc-cli/src/render.rs`
- [x] README: `ddc disk agents` section updated, with the "how to clean" wording. — done (PR #58): README `ddc disk agents`
- [ ] Cleanup half, after the deadline: the rule and ADR amendment of story "Coding agents' temporary folders outside the home", and the Mac app's rows in the "Coding agents" section.

## Notes

- 0.14.1 (PR #59) reads the temporary base only when `claude-<uid>` is a folder owned by the user, judged from its own metadata and never followed; otherwise `temp_root.refused` says why (`symlink`, `not-a-folder`, `not-owned`). `meta::uid` is that read gate only: it never feeds the removal guard, where `meta::owner` stays None on macOS (safety ADR amendment of 2026-10-31).

## Implementation Plan

Planner, 2026-11-01, against `main` at 1f3f408. Deadline MVP only.
1. `env.rs`: read `CLAUDE_CODE_TMPDIR` (and `TMPDIR` on Linux) in `from_process`, the one place that reads the environment; `Env::claude_temp_base()` resolves the override, else `<system_root>/private/tmp` on macOS or `$TMPDIR`/`<system_root>/tmp` on Linux, then `claude-<uid>`, canonicalised. A given `--system-root` wins over both variables. A hidden `--system-root` on `disk agents`, as `disk` has, and every existing agents test given a fixture root, so no test reads the real `/tmp`.
2. Model types in `agents/model.rs`, all optional; `AGENTS_SCHEMA_VERSION` stays 1.
3. Per-session byte counters in `agents/sizes.rs`, fed by the existing walk; the temporary base is walked as one more top-level folder but stays out of the harness's folders and bytes, which remain home-only.
4. `agents/footprint.rs`: the session ids (a UUID under `projects/<slug>/`), the join with `file-history/<id>` and `<temp base>/<slug>/<id>` by the id alone (the slug rule is not yet confirmed; a temporary folder whose transcript is gone keeps its own row), and liveness through the existing `liveness::claude`, from a metadata-only stat of each session's own files (not the walk's times, which a listing without times leaves empty and which then read as recent).
5. Wiring in `agents/mod.rs`, the regenerated schema, the text in `render.rs`, the tests, DDCKit's fixture, the README, a minor bump.

The rule "`claude-<uid>` is appended under `CLAUDE_CODE_TMPDIR` too" is an assumption until the research step confirms it; it lives in one pure function with its own test.

## Acceptance Criteria

- [x] On a synthetic home and system root, with two Claude Code sessions (one named by an injected live record, one with no record and files older than `entries::RECENT`) and a folder of the temporary base that no session owns (`bash-edit-diff`), `ddc disk agents --json` gives `sessions.footprints[]` one row per session with its transcript, checkpoint and temporary bytes and their total, marks the first live and the second not live, and `sessions.temp_root.bytes` minus the rows' `temp` equals the stray folder's bytes (test). — evidence: `crates/ddc-core/tests/agents.rs` `claude_footprint_per_session`; `crates/ddc-cli/tests/disk_agents_cli.rs` (schema contract), PR #58, reviewer
- [x] A session temporary folder whose transcript is already gone (Claude Code's own sweep) gets a row with `transcript: 0`, never folds into the remainder (test). — evidence: `agents.rs` `claude_footprint_per_session` (row C) and `footprint_without_a_transcript_and_under_a_skip`, PR #58
- [x] With `--system-root` given, the fixture wins over `CLAUDE_CODE_TMPDIR` and `TMPDIR`: no test reads the real temporary folder, even from a shell that sets them (test). — evidence: `disk_agents_cli.rs` (a `CLAUDE_CODE_TMPDIR` decoy loses to `--system-root`), `env.rs` unit tests, PR #58
- [ ] (Follow-up, after the research) A Codex thread's rollout file is attributed as its transcript, and a worktree inside a session's temporary folder is counted in `worktrees`, not twice (tests).
- [x] `--no-sizes` still finishes without measuring any folder (test asserts no size walk). — evidence: `crates/ddc-core/tests/agents.rs` `no_sizes_never_lists_the_temporary_folder` (op-stats directory-entry counter: 400 files in the temporary folder and 400 in `~/.claude/file-history` add no entry read under `--no-sizes`), PR #59, released in 0.14.1
- [x] The JSON validates against the updated schema, `schema_version` is unchanged, and the DDCKit contract test decodes it (test). — evidence: `committed_agents_schema_is_current`, `contract::holds("agents", ..)`, DDCKit `AgentsReportTests` (`app/Packages/DDCKit/Tests/DDCKitTests/AgentsTests.swift`), PR #58
- [x] No test or run reads the real `/tmp`, the real home, or a transcript's content (review, plus the test seam). — evidence: every agents test sets a fixture root (`with_system_root` / `--system-root`); `footprint.rs` reads listings and `symlink_metadata` only; reviewer, PR #58
- [ ] Manual (owner, reference Mac), recorded in the owner-checks runbook: the temporary bytes of all sessions plus `bash-edit-diff` equal `du -sk` of `/private/tmp/claude-$(id -u)`, or are within 1 % of it.

## Final Summary

## Technical Notes

- The liveness rule is the safety boundary. A live session's temporary folder and worktrees are reported as live and never described as cleanable.
- A worktree registered in a repository's `.git/worktrees` but missing on disk is not this story's. The worktree rules story ("worktree findings and provenance") owns orphans.

## Dependencies

- None blocking for the report half. The cleanup half needs story "Coding agents' temporary folders outside the home" (rule and ADR amendment).

## Attachments

-

---

*Last updated: 2026-11-01*

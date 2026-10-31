---
type: story
id: "story-coding-agents-temporary-folders-outside-the-home"
epic: "DDC-003"
title: "Coding agents' temporary folders outside the home"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-31
updated: 2026-10-31
external_refs: {}
tags: [agents, harness-data, temp, macos, linux, research]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Coding agents' temporary folders outside the home

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner ran `ddc disk agents` on the reference Mac (2026-10-31). It reported about 12 GiB across Claude Code, Codex, Cursor and opencode, all of it under the home folder. The owner noticed what it did not show: "where Claude keeps its worktrees and its work", under `/private/tmp`. On the same day `/private/tmp/claude-<uid>` held about 15 GB, more than everything the report found. `ddc` reads only home-folder paths (`agents::HARNESSES`), so this folder is invisible to both `ddc disk agents` and `ddc scan ~`.

That was deliberate. The agent ADR says "Session scratchpads under the system temp directory (`CLAUDE_CODE_TMPDIR`) lie outside `$HOME` and outside the default scan roots; the OS clears them, and `ddc` does not look for them". The safety ADR's 2026-10-28 amendment says "Outside the home folder ddc still removes nothing by path". This story reverses both, for removal. The report half, which only looks, is DDC-003's story "Per-session footprint: what each agent session keeps in its transcripts, temporary folders and worktrees". This story is the cleanup half, after the DEV deadline. The report-side code (the seam, the schema, the text) lives in the per-session story. This story owns the research, the removal amendments, the rule and the app row.

What the OS does, checked on the reference Mac on 2026-10-31:
- a restart clears `/private/tmp`: no entry there predates `kern.boottime`;
- nothing sweeps it by age: there is no `periodic` daily job or LaunchDaemon for it.

So the folder accumulates between restarts. The 15 GB was 33 hours of uptime: 9.2 GB in one session folder, 6.1 GB in a live session's, two empty ones, and 23 MB in `bash-edit-diff`. The dead share at any moment can be small. The research note "Removal wherever a size is shown: coding agents, protected folders and the whole disk" says a restart clears what is older than three days; this evidence contradicts it, and the research step settles that row.

Where it is. Claude Code's documentation (environment variables, `CLAUDE_CODE_TMPDIR`) says it appends `/claude-{uid}/` to a base: `/tmp` on macOS, `os.tmpdir()` on Linux and Windows, or the `CLAUDE_CODE_TMPDIR` override. So:
- the canonical path is `/private/tmp/claude-<uid>` on macOS (`/tmp` is a relative symlink to `private/tmp`) and `<tmpdir>/claude-<uid>` on Linux;
- an override set in `~/.claude/settings.json` (`env`) or in managed settings is readable; one set only in a shell is not, and the report says so;
- only the current user's folder is read, through `getuid()`. `/private/tmp` itself is never listed, so other users' folder names never enter a report.

What the folder holds, as seen on the reference Mac (layout to confirm in the research step):

- `/private/tmp/claude-<uid>/<project-slug>/<session-id>/`, one folder per Claude Code session:
  - `scratchpad/`: a session's own temporary files. Agents put clones, release builds, worktrees and test trees here; one folder held a 3.4 GB release build;
  - `tasks/`: the output of background tasks and subagents;
  - `images/`: pasted images;
- `/private/tmp/claude-<uid>/bash-edit-diff/`, sized in the total only. Every other child that is not a session folder is neither an entry nor claimed, as for the transcripts rule.

Subagents write under their parent session's folder, so the parent's liveness covers them. `/clear` starts a new session id in the same process. The old folder then looks dead by its record while the process lives (the agent ADR's open question on `/clear`). The in-use check below is the second net for that case.

`<session-id>` is the same id as the session's transcript in `~/.claude/projects/<project>/`. So the liveness check that `ddc disk agents` already runs for transcripts can also tell a live session's temporary folder from a dead one's.

## Decomposition

- [ ] Extend the research note (Codex, Cursor and opencode: whether they keep temporary data outside the home; Linux `/tmp` as tmpfs and systemd-tmpfiles ageing; empty session folders; the sibling `claude-mcp-browser-bridge-<user>`). "Coding-agent harnesses on macOS: where their data lives and how to tell a live session". Run the critic on it. It covers:
  - where each harness keeps temporary data outside the home folder on macOS and Linux: Claude Code's `/tmp/claude-<uid>`, and whether Codex, Cursor and opencode use `/tmp` or `$TMPDIR` (`/private/var/folders/<id>/T`);
  - whether the folder location follows an environment variable, and whether `<project-slug>` and `<session-id>` are stable;
  - when the system removes these folders on its own: at a restart, by age through `periodic`, or never. Verify it; do not assume;
  - whether a dead session's `scratchpad` can hold a git worktree that is still registered in a repository's `.git/worktrees`. Moving such a worktree to the Trash leaves that registration behind, so the agent ADR's worktree rules (`git.worktree`, `git.worktree-orphan`) apply to it.
- [ ] The report half lives in story "Per-session footprint: what each agent session keeps in its transcripts, temporary folders and worktrees".
- [ ] Dated amendments, with the critic, quoting the sentences they replace: the agent ADR §3 (the line "`ddc` does not look for them"), and the safety ADR §5 and §6 together with its 2026-10-28 amendment ("Outside the home folder ddc still removes nothing by path"; the re-check "the path lies inside the scan root"; the `--items` refusal outside the roots). The amendment adds a rule `agent.claude-code-temp`:
  - its entries are the session folders, the same per-entry model as `agent.claude-code-transcripts`;
  - class `review`, method `trash`, `listed_only`, as for the other agent rules;
  - a live session's folder is never offered;
  - the scope reaches outside the home folder, which no rule does today. The amendment picks how: either the folder becomes a scan root, or the rule is anchored outside the roots with a named exception to the re-check and the `--items` refusal;
  - it is resolved at run time (platform temp base, plus `getuid()`, plus a settings-file override), because `Matcher::KnownPaths` holds only static home-relative strings;
  - `/tmp` and `/private/tmp` (macOS) and `/tmp` (Linux) join `PROTECTED_SYSTEM`, with the folder-list test;
  - three liveness nets for a session folder: the session record, open files under its prefix, and the in-use check (cwd, executable, arguments) for a build or dev server still running in a dead session's scratchpad;
  - a session folder that holds a registered git worktree is refused, or the amendment accepts a dangling `.git/worktrees/<name>`, and says which.
- [ ] DDCKit: `ReportScope` and the Settings roots learn the agents' temporary root, or the rule's exception, with a test. Otherwise a scan with that root makes the scope `.outside` and switches Clean off in the app.
- [ ] The Mac app's "Coding agents" section shows the row, as the other harness folders are shown.
- [ ] Tests on a synthetic `/tmp`-like root passed in through a test seam. Never the real `/private/tmp`.

## Implementation Plan

## Acceptance Criteria

- [ ] The research note is extended and critic-approved.
- [ ] On a synthetic temporary root passed through a named seam (a hidden `--system-root` on `disk agents`, or a core test through `Env::with_system_root`), with liveness from injected records and never the test's own pid: the rule plans the dead session's folder and not the live one (test).
- [ ] The amendments are critic-approved. `clean --items` with the dead folder listed plans it as a `review` entry; with the live folder listed it refuses with `entry-locked live-session`; `--yes` selects neither; `--agent` skips it as `not-consented` (tests).
- [ ] After a scan on a Mac where the folder exists, the Review window's scope is `.same` or `.inside` and Clean stays enabled (DDCKit test).
- [ ] Manual (owner, reference Mac): `ddc disk agents` shows a Claude Code temporary row whose size equals `du -sk /private/tmp/claude-$(id -u)` or is within 1 % of it; it is recorded in the owner-checks runbook.

## Final Summary

## Technical Notes

- A coding-agent session must never lose its scratch space while it runs. The liveness rule is the safety boundary here, as it is for transcripts.
- The reference Mac's numbers above come from the owner's own run on 2026-10-31. They describe one machine, not a typical one.

## Dependencies

- Story "Per-session footprint: what each agent session keeps in its transcripts, temporary folders and worktrees" (the report half).
- None else blocking. It builds on DDC-003's story "Harness data rules" (done) and the `disk agents` liveness code.

## Attachments

-

---

*Last updated: 2026-10-31*

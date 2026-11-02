---
type: story
id: "story-coding-agents-on-linux-ddc-disk-agents"
epic: "DDC-010"
title: "Coding agents on Linux: ddc disk agents"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-02
external_refs: {}
tags: [linux, agents, capability, cli]
code_refs: []
specs: []
started_at: null
closed_at: "2026-11-02T14:25:44.227Z"
plan_updated_at: null
---

# Coding agents on Linux: ddc disk agents

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | done |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

`ddc disk agents` lists each coding-agent harness, its folders, its sizes and its live sessions. On Linux it refuses with exit 69, because Linux does not declare `coding-agents`. The pieces that guard a clean already run on Linux:
- the coding agents' catalog rules (`agent.*`);
- the plan's `agent-live` check;
- `agents/running.rs`, which matches harnesses by executable name;
- the `liveness.rs` unit tests.

What is missing is the report, and the harness folder table it reads.

`agents::HARNESSES` comes from the research note "Coding-agent harnesses on macOS: where their data lives and how to tell a live session". One of its entries is a macOS path, `Library/Application Support/Claude/claude-code`. The others (`.claude`, `.codex`, `.cursor`, `.local/share/opencode`, `.cache/codex-runtimes`) are the same on Linux, but nobody has checked where each harness keeps its data there.

The Linux research note's §6 found `~/.local/share/claude/versions` on Linux holding two Claude Code builds (474 MB). No harness entry and no rule covers it, on either platform. Its Conclusion 9 put that on the coding agents' data, on both OSes.

Claude Code's temporary folder already resolves per OS (`Env::claude_temp_base`: `CLAUDE_CODE_TMPDIR`, else `/private/tmp` on macOS, else `TMPDIR` and then `/tmp` on Linux), and `temp_base` refuses a folder in a shared `/tmp` that the user does not own. The report around it is what stays gated.

## Decomposition

- [x] Research addendum "On Linux" in the harness note, critic-approved (2026-11-02).
- [x] The safety ADR amendment of 2026-11-02: old builds as picked entries of their `versions` folder, critic-reviewed (the owner chose to let a single file be cleaned).
- [x] PR C1 (#93, 0.22.0), folding in the old items: `HARNESSES` resolved per OS, `coding-agents` declared on Linux (the agents report tests and the `disk agents` CLI tests leave `lacks(coding-agents)`, and the ADR "Platform seam" §3 Linux row is updated), and the cases flipping to run on Linux and validate against `docs/agents.schema.json`. Also the Linux liveness fixes (`pidDomain` per OS, a native Claude Code process told by its resolved executable, a fresh record of a pid this machine does not see is undecidable), `HARNESSES` per OS, and `coding-agents` declared on Linux.
- [x] PR C2, landed as C2a (#97, 0.22.4: the platform calls and the in-use fix) and C2b (#99, 0.23.0: the entries, locks, schemas and app): the old-builds entries for Claude Code and Cursor's agent, as the amendment states: the two `versions` folders as `review` findings with the `none` method, their builds as picked entries to the Trash only, the locks `build-in-use` and `builds-unknown` (scan schema raised), the new platform calls (resolved executable, other users' process names, `hidepid`, the home's filesystem type), and the resolved executable in the in-use check.

## Implementation Plan

## Acceptance Criteria

- [x] The research addendum exists with Linux paths for every harness in `HARNESSES`, critic-approved. — evidence: research note "Coding-agent harnesses on macOS…", section "On Linux (addendum, 2026-11-02)", critic passes 2026-11-02
- [x] `ddc disk agents --json` (`ddc agent status` once renamed) on Linux lists every harness found in a seeded home, with sizes and live sessions, and validates against `docs/agents.schema.json` (tests and cases). — evidence: crates/ddc-cli/tests/disk_agents_cli.rs `the_report_lists_the_harnesses_and_their_live_sessions` (contract `agents`), the case "disk agents lists no harness in a fresh home" now run on Linux; native Linux run 875/875 at C1; PR #93 checks
- [x] Claude Code's old builds (files) and Cursor's agent's (folders) are listed as entries on Linux and macOS. A build a launcher points to, a process runs, a lock names, or the newest, is locked `build-in-use`; each `builds-unknown` condition locks every build; none is ever selected by `--yes` (tests). — evidence: crates/ddc-core/src/agents/builds.rs `the_launchers_target_a_running_build_a_locked_one_and_the_newest_are_in_use`, `what_cannot_be_told_locks_every_build`; crates/ddc-core/tests/agent_builds.rs `old_builds_are_entries_and_the_ones_still_needed_are_locked`, `what_the_machine_cannot_tell_locks_every_build`, `a_build_that_came_into_use_after_the_plan_is_refused`; macOS and Linux CI on PR #99; main Gate run 41459741327 (b46a7ec), green
- [x] A picked build moves to the Trash through its verified parent, a file on both OSes, and the dry run predicts it (tests; on Linux through ddc's own Trash). — evidence: agent_builds.rs `a_picked_free_build_moves_to_the_trash_and_a_needed_one_is_refused`, `the_pruned_recheck_reads_the_other_builds_and_frees_an_old_one`; crates/ddc-core/tests/agent_builds_trash.rs (Linux, the real Trash: `files/` and `.trashinfo`); crates/ddc-cli/tests/clean_items_cli.rs `a_picked_old_build_is_predicted_and_permanent_is_refused`; main Gate run 41459741327 (b46a7ec), green
- [x] `CAPABILITIES` for Linux holds `coding-agents`, and `.config/native-skips/linux.txt` no longer lists the agents report tests. — evidence: crates/ddc-core/src/platform/linux/mod.rs (`CodingAgents`); linux.txt drops the 13 agents tests (PR #93); ADR "Platform seam" §3 Linux row, 2026-11-02

## Final Summary

**What changed.**
- **Research.** An "On Linux" addendum gives every harness's Linux paths and the XDG variables that move them. The old-builds design is in a safety ADR amendment the owner chose on 2026-11-02 (a single file may be cleaned). Both are critic-approved.
- **PR C1 (#93, 0.22.0).** `ddc agent status` runs on Linux:
  - harness folders per OS;
  - a session record is local by its `pidDomain`;
  - a native Claude Code process is told by its build folder;
  - on Linux a fresh record ddc cannot show to be live is `unknown`, which covers containers and WSL.
- **PR C2a (#97, 0.22.4).**
  - `ProcInfo.resolved_exe`, and the in-use check reads it, so on macOS a program run through a symlink is now seen.
  - macOS `process_start` for other users.
  - Other users' process names (macOS short BSD info), `hidepid`, a network home.
- **PR C2b (#99, 0.23.0).** Claude Code's build files and Cursor's agent's build folders become picked entries of two `review`/`none` findings:
  - Trash only, never `--yes` or `--permanent`;
  - locks `build-in-use` (launcher, process, lock, newest) and `builds-unknown` (six causes);
  - origin kind `builds`;
  - scan 6, clean 9, journal 9, progress 7;
  - the app's builds rows.

**Why.** Linux had no agents report, its session records all read as unknown, and old agent builds had no way out but by hand.

**Tests executed.**
- Native Linux runs: 875/875 at C1, 884 at C2a, 898 at C2b.
- CI on every PR: macOS, Linux, Windows check, app (Swift), linux-mounts.
- Main's full Gate green on b46a7ec (run 41459741327): the macOS goldens and the three pinned load-tree hashes, recorded on the Mac under the owner's exception.

**Risks and follow-ups.**
- On Linux the scan reads a file's birth time but the guard's `child_times` does not. A build or a user-data entry under 10 minutes old is predicted `would-trash` and then refused as `recent` at the clean. This fails safe; it is a follow-up for story "Linux file kinds…" or its own.
- The desktop app's builds folder and Windows's slot have no entries yet.
- The owner checks are in the runbook.

## Technical Notes

- **Deviations:**
  - The freshness check uses the injected clock and reads "within 10 minutes either way".
  - Unread build facts lock as `process-withheld`.
  - A bare-version process name counts as Claude Code (macOS `p_comm` may be the build's version), which fails closed.
  - The Linux real-Trash test runs as a child process of its test binary with `HOME` set, since `set_var` needs `unsafe`.
- **Mac exceptions the owner granted on 2026-11-02:** `make golden-record`, schema regeneration, `ddc-loadgen manifest` for the pinned hashes, and `make ddckit-test` for Swift-only pushes. All other builds run on the Windows host.

- In a container or under WSL, the harness and `ddc` may see different `/proc` views. The liveness check reads only one's own processes, as the safety ADR's amendment of 2026-10-31 states for Linux.
- Story "Per-session footprint: transcripts, temporary folders and worktrees" (DDC-003) is in progress and touches the same report. Rebase on it, and do not duplicate its session table.

## Dependencies

- DDC-003's harness research note and its per-session footprint story.
- Story "Cache roots moved by the environment: XDG and each tool's own variable", if a harness honours `XDG_DATA_HOME`.

## Attachments

-

---

*Last updated: 2026-11-02*

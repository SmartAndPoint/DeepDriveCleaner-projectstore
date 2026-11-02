---
type: story
id: "story-coding-agents-on-linux-ddc-disk-agents"
epic: "DDC-010"
title: "Coding agents on Linux: ddc disk agents"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-02
external_refs: {}
tags: [linux, agents, capability, cli]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Coding agents on Linux: ddc disk agents

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | planned |
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
- [ ] PR C1, folding in the old items: `HARNESSES` resolved per OS, `coding-agents` declared on Linux (the agents report tests and the `disk agents` CLI tests leave `lacks(coding-agents)`, and the ADR "Platform seam" §3 Linux row is updated), and the cases flipping to run on Linux and validate against `docs/agents.schema.json`. Also the Linux liveness fixes (`pidDomain` per OS, a native Claude Code process told by its resolved executable, a fresh record of a pid this machine does not see is undecidable), `HARNESSES` per OS, and `coding-agents` declared on Linux.
- [ ] PR C2: the old-builds entries for Claude Code and Cursor's agent, as the amendment states: the two `versions` folders as `review` findings with the `none` method, their builds as picked entries to the Trash only, the locks `build-in-use` and `builds-unknown` (scan schema raised), the new platform calls (resolved executable, other users' process names, `hidepid`, the home's filesystem type), and the resolved executable in the in-use check.

## Implementation Plan

## Acceptance Criteria

- [x] The research addendum exists with Linux paths for every harness in `HARNESSES`, critic-approved. — evidence: research note "Coding-agent harnesses on macOS…", section "On Linux (addendum, 2026-11-02)", critic passes 2026-11-02
- [ ] `ddc disk agents --json` (`ddc agent status` once renamed) on Linux lists every harness found in a seeded home, with sizes and live sessions, and validates against `docs/agents.schema.json` (tests and cases).
- [ ] Claude Code's old builds (files) and Cursor's agent's (folders) are listed as entries on Linux and macOS. A build a launcher points to, a process runs, a lock names, or the newest, is locked `build-in-use`; each `builds-unknown` condition locks every build; none is ever selected by `--yes` (tests).
- [ ] A picked build moves to the Trash through its verified parent, a file on both OSes, and the dry run predicts it (tests; on Linux through ddc's own Trash).
- [ ] `CAPABILITIES` for Linux holds `coding-agents`, and `.config/native-skips/linux.txt` no longer lists the agents report tests.

## Final Summary

## Technical Notes

- In a container or under WSL, the harness and `ddc` may see different `/proc` views. The liveness check reads only one's own processes, as the safety ADR's amendment of 2026-10-31 states for Linux.
- Story "Per-session footprint: transcripts, temporary folders and worktrees" (DDC-003) is in progress and touches the same report. Rebase on it, and do not duplicate its session table.

## Dependencies

- DDC-003's harness research note and its per-session footprint story.
- Story "Cache roots moved by the environment: XDG and each tool's own variable", if a harness honours `XDG_DATA_HOME`.

## Attachments

-

---

*Last updated: 2026-11-01*

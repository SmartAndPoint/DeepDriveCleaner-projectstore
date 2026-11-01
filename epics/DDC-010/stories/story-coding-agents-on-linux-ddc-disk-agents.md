---
type: story
id: "story-coding-agents-on-linux-ddc-disk-agents"
epic: "DDC-010"
title: "Coding agents on Linux: ddc disk agents"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
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

- [ ] Research: an addendum to the harness research note, "On Linux". For Claude Code, Codex, Cursor's agent CLI and opencode it gives:
  - where each keeps transcripts, sessions, builds, temporary files and worktrees on Linux;
  - whether XDG variables move them;
  - whether Claude Code's native installer prunes `~/.local/share/claude/versions` itself (an open question of the Linux research note);
  - measurements from one Linux machine with these harnesses in use, written without describing the machine.
  
  Run the critic on the addendum.
- [ ] `HARNESSES` resolves its folders per OS, as the shared catalog resolves rules (`Matcher` per-OS paths). Claude Code gains `~/.local/share/claude` (builds) on Linux and macOS, if the addendum confirms it on macOS.
- [ ] A rule, or a harness folder entry, for Claude Code's old builds in `versions`.
  - It keeps the build the `claude` launcher points to, and any build a live process runs (`/proc/*/exe`).
  - It keeps the newest one, like `NEWEST_KEPT` in `rules/toolchain.rs`.
  - Class `safe` if the addendum shows the installer re-downloads a missing build. Otherwise class `review`.
- [ ] Declare `coding-agents` on Linux. Move the `agents.rs` report tests and the `disk agents` CLI tests off `lacks(coding-agents)`, and update the Linux row of ADR "Platform seam" §3.
- [ ] Cases table: `ddc disk agents` and `ddc disk agents --json` run on Linux and validate against the disk schema. The runner derives the exit-69 expectation from `CAPABILITIES`, so declaring the capability is what flips these cases. Check that they now pass on Linux.

## Implementation Plan

## Acceptance Criteria

- [ ] The research addendum exists with Linux paths for every harness in `HARNESSES`, critic-approved.
- [ ] `ddc disk agents --json` on Linux lists every harness found in a seeded home, with sizes and live sessions, and validates against `docs/disk.schema.json` (tests and cases).
- [ ] Claude Code's old builds in `~/.local/share/claude/versions` are found on Linux and macOS. The build in use and the newest build are never planned (tests).
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

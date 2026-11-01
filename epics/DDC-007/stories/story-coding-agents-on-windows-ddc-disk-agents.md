---
type: story
id: "story-coding-agents-on-windows-ddc-disk-agents"
epic: "DDC-007"
title: "Coding agents on Windows: ddc disk agents"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [rust, windows, agents, capability]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Coding agents on Windows: ddc disk agents

| Field | Value |
|---|---|
| **Epic** | [DDC-007](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

`ddc disk agents` on Windows, so the Windows app can show the coding agents section (DDC-008 story "Windows app: Whole disk and coding agents views"). It is the Windows counterpart of DDC-010 story "Coding agents on Linux: ddc disk agents": the report exists, and what is missing is where each harness keeps its data on Windows and how a live session is told there.

## Decomposition

- [ ] Research: an addendum "On Windows" to the harness research note. For Claude Code, Codex, Cursor's agent CLI and opencode: where each keeps transcripts, sessions, builds, temporary files and worktrees under `%USERPROFILE%`, `%APPDATA%` and `%LOCALAPPDATA%`, and how a live session shows (process image and command line, which the Windows in-use guard of story "Windows scanner and rule catalog" reads). Measured on the owner's lab machine, written without describing it. Critic-reviewed.
- [ ] `HARNESSES` resolves its folders per OS for Windows.
- [ ] Declare `coding-agents` on Windows; move the report and CLI tests off `lacks(coding-agents)` there; amend the Windows row of ADR "Platform seam" §3.

## Implementation Plan

## Acceptance Criteria

- [ ] The addendum exists, critic-approved.
- [ ] `ddc disk agents --json` runs on Windows and validates against the disk schema (CI, test home).
- [ ] A live harness process in a test keeps its session out of the plan of a `clean --items` that names it (the `agent-live` check).

## Dependencies

- Story "Windows scanner and rule catalog" (the processes' image and command line).

## Final Summary

## Attachments

-

---

*Last updated: 2026-11-01*

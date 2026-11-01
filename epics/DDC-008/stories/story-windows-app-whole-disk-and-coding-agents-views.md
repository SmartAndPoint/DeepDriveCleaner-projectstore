---
type: story
id: "story-windows-app-whole-disk-and-coding-agents-views"
epic: "DDC-008"
title: "Windows app: Whole disk and coding agents views"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [windows, tauri, app, disk, agents]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Windows app: Whole disk and coding agents views

| Field | Value |
|---|---|
| **Epic** | [DDC-008](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The macOS Whole disk window (`DiskWindowController`, `DiskView`, opened from the popover's "Where Does the Rest Go?", with `AreaActionSheet`, `CheckedFree` and Show in Explorer) and the panel's coding agents section (`AgentsSection`) on Windows ([[adr/windows-app-tauri-2-shell-over-the-ddc-cli|Windows app: Tauri 2 shell over the ddc CLI]]). The views are driven by the CLI's declared capabilities: each appears only when `rules --json` lists `whole-disk` or `coding-agents`. Until the Windows CLI declares them (DDC-007 stories "Whole disk on Windows: the areas outside the home" and "Coding agents on Windows: ddc disk agents"), the panel shows neither, and nothing in the app pretends otherwise.

Homebrew (`disk brew-packages`) has no Windows counterpart and is not part of this story.

## Decomposition

- [ ] `ddc-app-model`: `DiskReport`, `DiskSummary`, `DiskModel`, `AreaActions`, `DiskFreeRun`, `FreeReport`, `AgentsSummary`, ported with their tests; the lane rules for `disk` and `disk agents` (macOS ADR §1, amended).
- [ ] The Whole disk window and the panel's agents section, shown by capability; `disk free --actions <file> --json` only after its `--dry-run` preview and confirmation, with each action's exact command and what it cannot undo on the sheet (macOS ADR §7, amended).

## Implementation Plan

## Acceptance Criteria

- [ ] A model test shows each view hidden when its capability is absent and shown when present.
- [ ] `disk free --actions` runs only after the confirmation (the argv allowlist test extends to it).
- [ ] With the capabilities declared on Windows: manual check by the owner on the lab machine.

## Dependencies

- Story "Windows app: Tauri shell, tray icon and the panel".
- DDC-007 stories "Whole disk on Windows: the areas outside the home" and "Coding agents on Windows: ddc disk agents" for the views to show anything.

## Final Summary

## Attachments

-

---

*Last updated: 2026-11-01*

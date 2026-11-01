---
type: story
id: "story-whole-disk-on-windows-the-areas-outside-the-home"
epic: "DDC-007"
title: "Whole disk on Windows: the areas outside the home"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [rust, windows, disk, capability]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Whole disk on Windows: the areas outside the home

| Field | Value |
|---|---|
| **Epic** | [DDC-007](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

`ddc disk` on Windows: the areas of the system drive outside the home, so the Windows app can show its Whole disk view (DDC-008 story "Windows app: Whole disk and coding agents views"). The Windows counterpart of DDC-010 story "Whole disk on Linux: the areas outside the home".

Candidate areas, to be confirmed by measurement: `Windows` (with `WinSxS`, which only DISM may clean), `Program Files` and `Program Files (x86)`, `ProgramData`, other users' profiles (counted, not sized: a standard user cannot read them, so they are reported as not readable), the page file, the hibernation file and the swap file, the WSL distributions' and Docker Desktop's disk images, the Recycle Bin. As on macOS, ddc reports what owns each area and which tool frees it; it never deletes outside the home itself.

`disk free` actions on Windows: the conda actions where conda is installed. Any other action (for example a Docker or WSL compaction) needs its own decision in the safety ADR first.

## Decomposition

- [ ] Research: an addendum "On Windows" to the whole-disk research, measured on the lab machine, critic-reviewed.
- [ ] The Windows areas in `disk::Sources` and the area table.
- [ ] Declare `whole-disk` on Windows; tests off `lacks(whole-disk)` there; ADR "Platform seam" §3 amended.

## Implementation Plan

## Acceptance Criteria

- [ ] `ddc disk --json` runs on Windows and validates against the disk schema.
- [ ] No area outside the home is opened for writing; a test lists the `disk free` actions allowed on Windows.

## Dependencies

- Story "Windows scanner and rule catalog".

## Final Summary

## Attachments

-

---

*Last updated: 2026-11-01*

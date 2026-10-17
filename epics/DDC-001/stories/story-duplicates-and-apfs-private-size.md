---
type: story
id: "story-duplicates-and-apfs-private-size"
epic: "DDC-001"
title: "Duplicates and APFS private size"
status: planned
priority: p3
assignee: "Evgenii Konev"
created: 2026-10-17
updated: 2026-10-17
external_refs: {}
tags: [dupes, apfs]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Duplicates and APFS private size

| Field | Value |
|---|---|
| **Epic** | [DDC-001](../epic.md) |
| **Status** | planned |
| **Priority** | p3 |
| **Assignee** | Evgenii Konev |

---

## Description

Two improvements on top of the main scanner. The first is duplicate search in user data (photos, downloads, videos), report only. The second is the exact size of the **union** of selected findings with APFS clones. Private size (from the first version it is measured only in findings where clones are expected: uv, pnpm, bun) is a lower bound for a single finding: if both a `.venv` and the uv cache are selected, both sides of the clone show ≈ 0, although the full size will be freed (see [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]]).

## Decomposition

- [ ] `ddc dupes [PATH...]`: group by size → hash of the first 64 KB → full hash (blake3); clones of the same file (`ATTR_CMNEXT_CLONEID`) are not counted as duplicates
- [ ] Report: duplicate groups, how much will be freed, which copy is older
- [ ] Union count: in the selected set, each `ATTR_CMNEXT_CLONEID` is counted once at full size; for files whose clones are outside the selection, the private size is used
- [ ] The `clean` plan shows the exact "frees" value for the selected set

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test: two identical files and one that differs in the last byte — exactly one group is found
- [ ] Test with `cp -c`: selecting the original and the clone together shows the full size; selecting each one alone shows the private size
- [ ] On the reference machine, selecting "all `.venv` + uv cache" shows more than the sum of private sizes (summary)

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- `ATTR_CMNEXT_CLONEID` is available on APFS with `VOL_CAP_FMT_CLONE_MAPPING` support; without it, only the lower bound remains.
- Duplicates are report only; deletion is manual (class `user-data`).
- The extension point already exists: `DirReader` and the scanner's `on_file` hook.

## Dependencies

- [[epics/DDC-001/stories/story-workspace-skeleton-and-parallel-fs-scanner|Workspace skeleton and parallel filesystem scanner]]

## Attachments

-

---

*Last updated: 2026-10-17*

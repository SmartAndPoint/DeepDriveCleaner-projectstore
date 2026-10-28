---
type: story
id: "story-load-run-on-the-reference-machine-including-system-folders"
epic: "DDC-005"
title: "Load run on the reference machine, including system folders"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [rust, performance, benchmark, release]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Load run on the reference machine, including system folders

| Field | Value |
|---|---|
| **Epic** | [DDC-005](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

Generated trees model real disks, but they are not real disks. The `aa64c83` steps were found on the reference machine's system trees. DDC-001's success criteria are also stated as reference-machine numbers: `ddc scan ~` ≤ 60 s, `ddc scan ~/Projects` ≤ 20 s. Nothing re-measures them after each release.

This story adds `make bench-real`. It is a read-only run of the release binary on the reference machine. It records time, peak memory and CPU for each scan, and compares them with the last run. It is advisory: the owner runs it at their own pace, and it does not block a release.

## Decomposition

- [ ] `make bench-real` runs the release build three times after one warm-up, and records the median of:
  - `ddc scan ~ --json`;
  - `ddc scan ~/Projects --json`;
  - `ddc disk` (areas outside the home, from story "Whole disk: what lies outside the home folder" in DDC-002).
- [ ] It records only aggregate columns, the same as `make load`: wall, user and system time, peak RSS, threads, entries, total bytes found. No paths, folder names or findings go into the file, because `bench/` is in a public repository.
- [ ] It runs the comparison script and prints what changed.
- [ ] Read-only: no `clean`. `--data-dir` points to a scratch folder, so the run leaves the owner's last scan and settings untouched.
- [ ] Add a line to the release notes procedure: "run `make bench-real` when convenient after a release; it is advisory".

## Implementation Plan

## Acceptance Criteria

- [ ] `make bench-real` exists and writes `bench/real-<date>.tsv` with aggregate columns only (evidence: one committed run, checked to hold no path).
- [ ] The run meets DDC-001's limits (home ≤ 60 s, `~/Projects` ≤ 20 s), or the story records the miss and opens a follow-up.
- [ ] Manual (owner): the run is done from the owner's terminal on the reference machine, with Full Disk Access as in daily use. Agents do not run it, because system areas can raise permission prompts.

## Final Summary

## Technical Notes

- A run from an agent session can hit TCC prompts or read areas the owner has not granted. This run is the owner's, by design.
- Numbers depend on what is on the disk that day. The comparison flags changes. It does not judge them, because the disk changed too. The entries column helps tell the two apart.

## Dependencies

- Story "Memory and CPU under load: peak memory, leaks and CPU time" (the harness, columns and comparison script).

## Attachments

-

---

*Last updated: 2026-10-28*

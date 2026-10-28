---
type: story
id: "story-rule-catalog-split-shared-and-per-os-sections"
epic: "DDC-007"
title: "Rule catalog split: shared and per-OS sections"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [rust, rules, portability]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Rule catalog split: shared and per-OS sections

| Field | Value |
|---|---|
| **Epic** | [DDC-007](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

`rules/catalog.rs` (631 lines) mixes rules that hold everywhere with rules that name macOS paths:
- everywhere: `node_modules`, `target`, `.venv`, `~/.cargo/registry`, `~/.npm`, `~/.gradle`, `~/.m2`;
- macOS only: `~/Library/Caches/…`, Xcode, simulators.

The protected-folder list of the safety ADR §6 is also macOS-shaped. This story splits both into a shared section and per-OS sections, so the Linux and Windows stories add rules without touching macOS. The macOS catalog and its output do not change.

## Decomposition

- [ ] Classify every rule: shared (same relative path everywhere), shared with per-OS paths (one rule id, a path per OS, e.g. pip's cache), or OS-only.
- [ ] The catalog becomes `rules/catalog/{shared,macos,linux,windows}.rs`. Linux and Windows start empty except the shared rules. Rule ids are unchanged, so settings that disable rules keep working.
- [ ] The protected-folder list splits the same way. The safety ADR §6 list stays the macOS list.
- [ ] `ddc rules` prints the rules of the running platform, and `--platform <os>` prints another platform's, for review.
- [ ] Golden test: `ddc rules --json` and `scan --json` on all S profiles are identical on macOS before and after.

## Implementation Plan

## Acceptance Criteria

- [ ] On macOS, `ddc rules --json` and the S-profile scans are identical before and after (`make golden-check` from DDC-005 story "Synthetic load trees: large, wide, deep and hostile").
- [ ] Every rule id appears in exactly one section, or in the shared section with per-OS paths (test).
- [ ] `ddc rules --platform linux` and `--platform windows` list the shared rules with their per-OS paths.

## Final Summary

## Technical Notes

- This also helps DDC-006: `catalog.rs` is one of the four files over 500 lines, and the split by platform and category removes it from the shape baseline.

## Dependencies

- Story "Move OS calls behind the platform module".

## Attachments

-

---

*Last updated: 2026-10-28*

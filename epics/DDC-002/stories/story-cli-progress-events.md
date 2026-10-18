---
type: story
id: "story-cli-progress-events"
epic: "DDC-002"
title: "CLI: progress events"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-18
external_refs: {design_canvas: "https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3"}
tags: [cli, contract, json, ux]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# CLI: progress events

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The app shows a scan's and a clean's progress from the CLI's machine-readable events. This story adds `--progress json` (§6 item 3 of the ADR "Menu bar app: Swift shell over the ddc CLI").

The clean's counter is built by the DDC-001 story "Progress output while ddc clean runs". This story only emits it as JSON, and cannot start before that story is done.

## Decomposition

Rough until the plan gate.

- [ ] `--progress json` for `scan` and `clean`: newline-delimited events on stderr, each with `schema_version`, documented in `docs/progress.schema.json`
- [ ] Scan events: files, directories and bytes so far
- [ ] Clean events: the preparing phase with the files walked; then item `i` of `n` with its path (or the native command's name), its outcome and the bytes processed
- [ ] Events are throttled: at most one every 100 ms, plus a final one
- [ ] Without `--progress json`, the terminal spinner and the progress line behave as before

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test: the events of a scan of a temp tree validate against `docs/progress.schema.json`, and the final event's counts equal the report's totals
- [ ] Test: `clean --items` over a temp tree emits the preparing phase, then one item event per item, the last with `i` equal to `n`
- [ ] Test with an injected clock (or a fake walker that yields slowly): over a run whose events would come every 10 ms without throttling, no two emitted events are less than 100 ms apart, except the final one, and more than one event is emitted, so the test cannot pass vacuously
- [ ] Test: without `--progress json`, a piped run writes no JSON to stderr (the assertion of "Progress output while ddc clean runs" still holds)
- [ ] `make check` passes

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- The counter lives in the core and is shared by the terminal line and the JSON events, so both read the same snapshot.
- The app runs the CLI it ships with; the schema still guards scripts.

## Dependencies

- [[epics/DDC-001/stories/story-progress-output-while-ddc-clean-runs|Progress output while ddc clean runs]] (DDC-001): hard dependency, the counter
- [[epics/DDC-002/stories/story-cli-clean-an-explicit-item-list|CLI: clean an explicit item list]]
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]

## Attachments

- Design canvas: https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3

---

*Last updated: 2026-10-18*

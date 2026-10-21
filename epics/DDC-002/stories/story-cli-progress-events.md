---
type: story
id: "story-cli-progress-events"
epic: "DDC-002"
title: "CLI: progress events"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-21
external_refs: {design_canvas: "https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV"}
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

### Revision after the design review (2026-10-21)

Artboard 11 of the design canvas ("Clean progress") draws the determinate left-to-right bar the owner asked for, in place of the sliding one. Reviewing it fixed what the events must carry.

**The clean's real sequence** is waiting → preparing → cleaning → stopping (`ReviewModel.Phase`); `.preparing` covers the CLI's rescan and then its keep-marker walk.
- Step 1, "Checking again", is that phase, with two counters. The rescan counts files read against the last report's `totals.files`, an estimate if the roots changed. The `keep_markers` walk counts files walked against the listed findings' `files`. Today that walk takes a throwaway `Progress::default()` (`crates/ddc-core/src/exec/mod.rs:445`), so it must report into the clean's progress.
- Step 2 is the item loop.

**Each item event carries:**
- `i` of `n`;
- its path, or the tool's name;
- `action` and `outcome`;
- its `bytes`.

An item's bytes count as processed whatever the outcome; the app adds "· N skipped".

**The bar.**
- **Basis:** bytes processed, with items as the alternative.
- **Item in flight:** shown as a striped segment, because there is no progress inside an item.
- **Native-only item:** takes its share of items, never bytes. Its space is not in the total (safety ADR §5), and the sheet says that what Docker frees inside its disk image is not counted. The bar's byte total therefore leaves native-only items out. Today `Confirmation.bytes` and `selectedReclaimable` include them (`Confirmation.swift:64`, `ReviewModel.swift:333`), so the sheet's header total changes with this story, or the bar never reaches 100 %.

**Stop.**
- Between file items, the clean stops after the current one.
- A running native tool is interrupted: SIGTERM goes to its process group (story "CLI: clean an explicit item list"). The sheet says so while a tool runs.
- The button reads Stop, with Esc (story "HIG pass: keyboard, colour, empty states and menus").

The Decomposition and ACs below are re-cut at the plan gate to carry these: per-item `action`, `outcome` and `bytes`, and the two counters of the preparing phase in the schema and in the `clean --items` AC.

## Decomposition

Rough until the plan gate.

- [ ] `--progress json` for `scan` and `clean`: newline-delimited events on stderr, each with `schema_version`, documented in `docs/progress.schema.json`
- [ ] Scan events: files, directories and bytes so far
- [ ] Clean events: the preparing phase with the files walked; then item `i` of `n` with its path (or the native command's name), its outcome and the bytes processed
- [ ] Events are throttled: at most one every 100 ms, plus a final one
- [ ] Without `--progress json`, the terminal spinner and the progress line behave as before
- [ ] The app side: `CLIRunner`'s stderr lines feed `ScanProgress` of "Menu bar popover", so the scanning state shows "N files read · T s" (artboard 1), and the clean progress of "Review and clean window" (item `i` of `n`, its path or the tool's name, the bytes processed); from this story on the app passes `--progress json` to `scan` and `clean --items`

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
- [ ] Unit test: scan progress events from a fake CLI update the popover's file count; a scan without events shows the elapsed time only
- [ ] Unit test: clean progress events from a fake CLI move the Review window's progress to item `i` of `n` with its path; a clean without events still shows its two phases
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
- [[epics/DDC-002/stories/story-review-and-clean-window|Review and clean window]]: the clean progress state this story feeds

## Attachments

- Design canvas: https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV

---

*Last updated: 2026-10-21*

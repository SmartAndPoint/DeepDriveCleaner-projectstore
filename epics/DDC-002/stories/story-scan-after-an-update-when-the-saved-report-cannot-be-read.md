---
type: story
id: "story-scan-after-an-update-when-the-saved-report-cannot-be-read"
epic: "DDC-002"
title: "Scan after an update when the saved report cannot be read"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-11-04
updated: 2026-11-04
external_refs: {}
tags: [app, scheduler, updates, popover]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Scan after an update when the saved report cannot be read

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner, 2026-11-04, on a screenshot of the popover after the update to 0.34.0: "No scan yet", "A scan reads your home folder and changes nothing", Scan Now. It happens after almost every update, since a report of another `schema_version` counts as no report (`PopoverModel`, since 0.14.10). Until a new scan, the popover, the Review window, the space band's baseline and "Grew since …" have nothing to show.

**Decided by the owner on 2026-11-04: the app scans by itself after an update when the saved report cannot be read.** This supersedes the paragraph "No automatic scan" of the story "Popover: after an update, say why a new scan is needed" (2026-11-02), whose wording work stays for what is left (scheduled scans off or paused, a report of a newer `ddc`).

It is not a third kind of scan. The objection of 2026-11-02 was that a self-started scan would run at full speed at relaunch and could raise the privacy prompts the background path avoids. So the scan after an update **is the scheduled scan, made due at once**:

- At the first launch of a new build (the last launched build in `AppPreferences` differs; written at every launch), when `last-scan.json` reads as `otherSchema` with an **older** version, and scheduled scans are on and not paused, the app sets its own due input, `UpdateScanDue` (a new `ScanDue.Inputs` field and `AppPreferences` key, beside the reminder slot, so a pending "Remind me tomorrow" slot of ADR §5 B7 is kept). The decision is awaited before the launch's tick, as `healing` is, in both modes: a normal launch runs one tick; a `--scheduled` launch's own tick runs it, with no second tick.
- The tick runs it through `ScheduledWork.tick()` and `popover.scheduledScan()`: `scan --background --notify json`, three threads, the folders macOS asks about skipped without Full Disk Access, the shared lock (a running clean keeps it due), the "update waits to install" hold (empty at a first launch; a later Update and Relaunch waits for the scan as today), the notification rules (a notification may follow right after the relaunch).
- **It is cleared when a readable report of the current schema is loaded**, whoever wrote it: a Scan Now before the tick leaves nothing due, so the person's scan is not followed by a second one. A `.busy` or `.failed` tick keeps it due, and the popover shows the wording of the other story with Scan Now.
- Turning scheduled scans off clears `UpdateScanDue`, as it clears the reminder slot and the pending notification (ADR §5 B9).
- While it runs, the popover says "Scanning after the update…" in place of "No scan yet"; `UpdateScanDue` is what tells it apart from a reminder or slot scan.
- **Not** when scheduled scans are off or paused (the person chose no scans without them; the popover keeps the wording of the other story), when the report is of a **newer** schema (a newer Homebrew `ddc` wrote it: the popover suggests updating the app), when there is no report at all (a first launch stays the person's Scan Now), or while the 0.1.0 agent exists (`legacyAgentFound`: nothing is due until Replace It, as today).
- Accepted costs: the scan sets `lastScheduledScan` to now, so a daily slot later the same day scans again; on a Mac without Full Disk Access the report leaves out Desktop, Documents and Downloads, as every scheduled scan does, and may show less than the hand-made report it replaces.

## Decomposition

- [ ] `AppPreferences`: the last launched build and `UpdateScanDue`; `ScanDue.Inputs`: the new input, due when set, cleared by the tick after a scan and by a loaded readable report
- [ ] DDCKit: the decision as a function of (build changed, report state older/newer/none/readable, schedule on, paused, legacy agent), awaited before the launch's tick
- [ ] `AppDelegate`/`LaunchMode`: one tick at a normal launch of a new build; none extra at a `--scheduled` launch
- [ ] `PopoverModel`/`PopoverView`: "Scanning after the update…" while that scan runs
- [ ] Menu bar ADR §5: a dated amendment naming `UpdateScanDue`: the scheduled scan made due at the first launch of a new build when the saved report is of an older schema
- [ ] DDCKit tests

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test (DDCKit): a new build, an older-schema report, scheduled scans on and not paused: `UpdateScanDue` is set, the launch's one tick runs the scheduled scan, the popover shows "Scanning after the update…" while it runs, then the new report, and nothing is left due
- [ ] Test (DDCKit): in `--scheduled` mode the launch's own tick runs the update scan, with no second tick
- [ ] Test (DDCKit): a pending reminder slot is kept and still runs at its time
- [ ] Test (DDCKit): a Scan Now before the tick leaves nothing due, and no second scan follows
- [ ] Test (DDCKit): nothing is set when the build is the same, when scheduled scans are off or paused, when the report is of a newer schema, when there is no report, when the report reads, or while the 0.1.0 agent exists; a clean holding the lock, or a failed scan, keeps it due and the popover shows the other story's wording with Scan Now
- [ ] `make ddckit-test` passes and the Gate's `app` job builds the app
- [ ] Menu bar ADR §5 carries the amendment; the critic has read it
- [ ] Owner check (runbook "Owner checks: manual acceptance across stories"): after the next update that changes the scan schema, the popover shows the scan running within a minute of the relaunch, then the new report; on a Mac whose app lacks Full Disk Access, no privacy prompt appears

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- The later scans need the agent's ticks: on the owner's Mac no tick has run since 2026-10-31 (story "Scheduled scans: register the agent again after an update"). The one tick at launch runs in the app itself and does not need launchd.
- `ScanDue` treats a reminder slot at or before now as due and the tick clears it; `UpdateScanDue` follows the same pattern as its own input, so a B7 reminder is never overwritten.
- The report state comes from `ScanReport.read` (`otherSchema(version)`, `PopoverModel.apply`); "older" compares that version with the app's `SCHEMA_VERSION`.
- The last launched build (this story, written at every launch) and the story "Scheduled scans: register the agent again after an update"'s `SchedulerRegistration` (written only while scheduled scans are on) are separate on purpose.

## Dependencies

- "Scheduled scans: register the agent again after an update" (DDC-002), first.
- Menu bar ADR "Menu bar app: Swift shell over the ddc CLI" §5.
- "Popover: after an update, say why a new scan is needed" (DDC-002): narrowed by this story.

## Attachments

-

---

*Last updated: 2026-11-04*

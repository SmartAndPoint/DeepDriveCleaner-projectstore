---
type: story
id: "story-popover-after-an-update-say-why-a-new-scan-is-needed"
epic: "DDC-002"
title: "Popover: after an update, say why a new scan is needed"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-02
updated: 2026-11-04
external_refs: {}
tags: [app, popover, updates]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Popover: after an update, say why a new scan is needed

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner, 2026-11-02, on a screenshot: after an update the popover often says "No scan yet" and "A scan reads your home folder and changes nothing", as on a first launch, although the Mac was scanned many times. Since 0.14.10 a `last-scan.json` written by an older schema counts as no report (`PopoverModel`, the "older schema" branch), which removed the error but kept the first-launch wording. A report schema changes often (scan 4 → 5 is planned with the Linux trash work), so this happens after most updates.

The popover should say what it knows, not guess. An older-schema report has two sources: an update of the app, or a `ddc` of another version (a Homebrew CLI, a legacy LaunchAgent) writing into the same data folder (`cmd/scan.rs`: "The app will show this report until its next scan"). So:

- **The text names the report's maker:** "Your last scan was made by ddc 0.17.4. This version reads reports differently, so it needs a new scan." It says "after the update" only when the report's `ddc_version` is older than the app's own version; when it is newer ("made by ddc 0.19.0"), it says so and suggests updating the app, as the unreadable-newer case does today.
- **Scan Now is the main action;** the line below gives the next scheduled scan as `ScanDue` computes it (which may be days away), not a fixed "within the hour".
- **The header subtitle** reads "Last scan by ddc 0.17.4 · scan again", not "No scan yet". The menu bar icon shows no warning, as since 0.14.10.
- **When the report's version cannot be read** (no `ddc_version` in the older file), the text omits it: "Your last scan was made by an earlier version of ddc".
- The Whole Disk view's "Scan Whole Disk Again" (0.14.10) uses the same wording family, from `last-disk.json`'s header.

*Superseded 2026-11-04 by the owner: the story "Scan after an update when the saved report cannot be read" makes the scheduled scan due at once after an update, when scheduled scans are on and not paused; this story keeps the wording for the cases left (scheduled scans off or paused, a report of a newer `ddc`).* **No automatic scan.** Considered and dropped: the menu bar ADR (§5, §6) knows two kinds of scan, a person's and the scheduled one, and a third, self-started kind would scan at full speed at relaunch and could raise the privacy prompts the background path avoids. The person starts it with Scan Now.

## Decomposition

- [ ] DDCKit: `VersionHeader` also decodes `ddc_version` (optional); `ScanReport.ReadError.otherSchema` carries it
- [ ] `PopoverModel`: a state for an older- or newer-schema report with the maker's version, `Content.otherVersion`, the header subtitle
- [ ] `PopoverView`: the block with Scan Now and the next scheduled scan from `ScanDue`
- [ ] `DiskModel`/`DiskView`: the same wording family from `last-disk.json`'s header
- [ ] DDCKit tests

## Implementation Plan

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test (DDCKit): a `last-scan.json` of an older schema whose `ddc_version` is older than the app's gives the other-version content, not `firstScan`; its text names that version and says "after the update"; the header subtitle is not "No scan yet"; the icon's attention list is empty; Scan Now is offered; the next-scan line comes from `ScanDue`
- [ ] Test (DDCKit): an older-schema report whose `ddc_version` is newer than the app's (a newer Homebrew CLI) does not say "after the update" and suggests updating the app
- [ ] Test (DDCKit): no `last-scan.json` at all still gives `firstScan` with today's wording
- [ ] Test (DDCKit): when the older report's `ddc_version` can be read, the text names it ("made by 0.17.4"); when it cannot, the text omits it
- [ ] Test (DDCKit): the Whole Disk view's outdated text names the update in the same words
- [ ] `make ddckit-test` passes and the Gate's `app` job builds the app
- [ ] The app starts no scan of its own when scheduled scans are off or paused, or the report is of a newer schema (the story "Scan after an update when the saved report cannot be read" covers the rest)
- [ ] Owner check (runbook "Owner checks: manual acceptance across stories"): after the next update that changes a report schema, the popover names the version that made the last scan and asks for a new one

## Final Summary

## Technical Notes

- `ScanReport.read` throws `otherSchema(version)` before decoding; reading `ddc_version` from an older file needs a small header decode (schema_version, ddc_version), as `VersionHeader` does for the schema version.

## Dependencies

- "Scan after an update when the saved report cannot be read" (DDC-002): it narrows this story to the cases left.
- Follows the 0.14.10 fix "a report from before an update asks for a new scan, not an error".

---
type: story
id: "story-automatic-cleaning-of-download-caches-after-a-scheduled-scan"
epic: "DDC-002"
title: "Automatic cleaning of download caches after a scheduled scan"
status: planned
priority: p3
assignee: "Evgenii Konev"
created: 2026-10-21
updated: 2026-10-21
external_refs: {}
tags: [app, safety, background]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Automatic cleaning of download caches after a scheduled scan

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p3 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner chose the strict variant of automatic cleaning on 2026-10-21. It is a third choice under "After a scheduled scan" in Settings → Automation (artboard 10 of the design canvas, story "Settings: folders, types, notifications"). The choices:
- Notify me (the default);
- Clean download caches automatically, then notify me;
- Only show it in the menu bar.

The automatic choice reverses a rejected alternative, so it waits for two ADR amendments before any code:
- **The safety ADR "Cleanup safety model: risk classes, plan and confirmation":**
  - Alternative A ("background mode cleans `safe` automatically"), rejected because the user chose "scan only", with "It can come back later as an option";
  - §8 ("the background path contains no call to the cleanup executor").
- **The menu bar ADR "Menu bar app: Swift shell over the ddc CLI":**
  - §7: the sheet's Clean is the only caller of a clean, which `ContractTests.cleanHasOneOwner` enforces;
  - §5: a scheduled instance lives about a minute, while a clean adds a rescan, the items and a rescan after it.

## Decomposition

Rough until the ADR amendments are accepted.

- [ ] Amend both ADRs, then run the critic.
- [ ] Scope:
  - only `safe` groups marked Auto in Types;
  - Auto is on by default only for "Package downloads" (npm, pnpm, yarn, bun, pip, Poetry, Homebrew, CocoaPods), after the `rules --json` group split of "Settings: folders, types, notifications";
  - it runs only when that frees at least N GiB of private bytes;
  - anything in use is skipped, as in 0.2.1.
- [ ] CLI: an unattended clean of the Auto groups, with `--json` and progress.
  - **Today.** `clean --yes` runs without a TTY but filters by `--class` and `--category`, not by group. `--json` requires `--items` (`crates/ddc-cli/src/cli.rs`), and no progress flag exists.
  - **Choice.** A new `--group` filter for `clean --yes`, with `--json`. `--items` stays a person's confirmation (menu bar ADR §7), so no machine-built item list is used. The §7 amendment rests on this choice.
- [ ] App:
  - the scheduled instance runs the clean after its scan;
  - a user launch that promotes it shows the clean with Stop;
  - the rescan after the clean updates `last-scan.json`.
- [ ] Working hours and the lock:
  - Outside working hours the automatic clean still runs; its notification waits for working hours like any other.
  - A Scan now pressed during a long scheduled clean waits behind the CLI lock, and the popover says so.
- [ ] Notification:
  - a "cleaned" category that says what was removed and what it freed, without Remind me tomorrow;
  - what `notify ack` records after an automatic clean.

## Implementation Plan

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Both ADR amendments are accepted after a critic pass, before any code.
- [ ] `ContractTests.cleanHasOneOwner` allows exactly one second owner: one file, one call site, the scheduled path.
- [ ] Test with a fake CLI: a scheduled instance with automatic cleaning on cleans only the Auto groups, and only above the floor. With it off, the scheduled instance runs no clean.
- [ ] Test: an item in use during the automatic clean is skipped and named in the notification's details.

## Final Summary

## Technical Notes

- **Owners.** Browser and app caches carry `owners`, so they are skipped while their app runs. That is why they are not Auto by default.
- **Where "never cleans" is restated.** Besides safety ADR §8, the rule that the background path never cleans appears in the safety ADR's 2026-10-18 and 2026-10-20 amendments and in the last line of menu bar ADR §7. The amendments must update all of them, so no contradiction is left.

## Dependencies

- "Settings: folders, types, notifications" (DDC-002): the Automation tab, the stored choice, and the split of the package-cache group.
- "Background scan and notifications in the app" (DDC-002): the scheduler, the `--scheduled` mode and notifications.

## Attachments

- Design canvas: https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV (artboard 10, Automation tab)

---

*Last updated: 2026-10-21*

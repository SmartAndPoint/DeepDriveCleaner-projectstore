---
type: story
id: "story-distribution-the-020-cask-with-the-app"
epic: "DDC-002"
title: "Distribution: the 0.2.0 cask with the app"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-18
external_refs: {design_canvas: "https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3"}
tags: [release, homebrew, distribution]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Distribution: the 0.2.0 cask with the app

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

DeepDriveCleaner 0.2.0 ships as one cask, `deepdrivecleaner`, with the app and the CLI linked from inside the bundle (ADR "Menu bar app: Swift shell over the ddc CLI", §8). It is signed by the existing team, 8MSDJTBW2D. The 0.1.0 release tooling is extended to a zipped, stapled app, and the reference machine is upgraded from 0.1.0.

## Decomposition

Rough until the plan gate.

- [ ] `make dist` builds the zipped, stapled app (story "App skeleton and build pipeline") and refuses a version mismatch between Cargo and the app's `MARKETING_VERSION`
- [ ] Extract the signing, notarization and retry helpers that `scripts/build-dist.sh` and `scripts/build-app.sh` each carry into `scripts/lib/release.sh`; one `make dist` run after the extraction shows the CLI tarball unchanged in contents, signature and notarization
- [ ] `scripts/publish-release.sh`:
  - asset `DeepDriveCleaner-<version>.zip`;
  - checks the app's signature, team and stapled ticket, and the embedded CLI's identifier and team;
  - keeps the 0.1.0 resume path; the tap is written through `gh api`
- [ ] Cask template:
  - `app "DeepDriveCleaner.app"`;
  - two `binary` stanzas from `#{appdir}/DeepDriveCleaner.app/Contents/Helpers/deepdrivecleaner`, one of them `target: "ddc"`;
  - `depends_on macos: :tahoe` (as GrammarHelper's cask) and `depends_on arch: :arm64`;
  - `uninstall quit: "com.smartandpoint.deepdrivecleaner"`;
  - `zap` the data, logs and preferences, with `launchctl:` for `com.smartandpoint.deepdrivecleaner` (0.1.0) and `com.smartandpoint.deepdrivecleaner.scheduler`;
  - caveats about Full Disk Access for the app
- [ ] Version 0.2.0 in the workspace and the app; release notes `docs/releases/0.2.0.md`; README leads with the app, keeps the CLI section, and says that `brew uninstall` without `--zap` can leave the app's entries in Login Items until macOS drops them
- [ ] Reference machine: `brew upgrade --cask deepdrivecleaner` from 0.1.0

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] `brew style` reports no offenses on the rendered cask; `brew audit --cask --online --strict` reports none on a machine with current Command Line Tools
- [ ] `scripts/publish-release.sh --dry-run` prints the rendered cask and notes; the 0.1.0 refusals still hold (dirty tree, HEAD not on `origin/main`, missing notes, stale build, an asset not signed by the configured team)
- [ ] Manual check on the reference machine: `brew upgrade --cask deepdrivecleaner` from 0.1.0 installs `/Applications/DeepDriveCleaner.app`, `ddc --version` prints 0.2.0, and `/opt/homebrew/bin/ddc` resolves into the bundle's `Contents/Helpers`. Full Disk Access is granted once, to the app
- [ ] Manual check: after `brew reinstall --cask deepdrivecleaner` (which quits the app), the next tick reopens the app, the scheduled scan runs, and the app's Full Disk Access grant still holds
- [ ] Manual check: `brew uninstall --zap --cask deepdrivecleaner` leaves neither agent: `launchctl print` finds no job for either label
- [ ] Manual check on a second Mac or a fresh user account: the downloaded app opens without a Gatekeeper warning (`spctl` accepted, Notarized Developer ID)
- [ ] Clean-machine access check on the owner's work laptop after `brew install --cask deepdrivecleaner` (owner's decision, 2026-10-18; it replaces the spike of "App skeleton and build pipeline"). Terminal has no Full Disk Access there, and the grant goes to DeepDriveCleaner.app only. Read from the app's own status and `~/Library/Logs/DeepDriveCleaner/app.log`:
  - before the grant the app shows no Full Disk Access; after it, a Scan now reports `full_disk_access: true`;
  - with no scheduled scan recorded yet, so the first tick is due: with the app quit, the tick launches it with `--scheduled`, and that scan reports `true`; a launch from Finder while that scan runs (about a minute on a full home) promotes the instance; a tick while the app runs normally is a reopen, with no second instance;
  - after `brew reinstall --cask deepdrivecleaner`, which removes the app and copies the new one, the grant still holds and the next tick opens the new app;
  - with the notification threshold in Settings below the scan's result, the scheduled scan's notification shows the app's name and icon.
  If a scheduled scan reports `full_disk_access: false` with the grant on the app, ADR item 5 and Rationale 1 are revised; the ADR has no fallback for that. If the grant does not survive the remove-then-copy, Rationale 1 and this cask are revised

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- A zip, not a DMG: the cask is the install path, and a zip of a stapled app also works when downloaded by hand.
- `uninstall quit:` runs on upgrade and reinstall too; the scheduler agent reopens the new app at the next tick, so scheduled scans continue.
- The CLI's path becomes stable inside `/Applications`, so the grant of a CLI started by launchd (`ddc schedule install`, for people who use only the CLI) survives upgrades.
- GrammarHelper's cask writes `depends_on macos: :tahoe`; current Homebrew reads a bare symbol as that version or later, and `brew style` decides the accepted form.
- The team switch is not part of this release: "Signing team: parameterise TEAM_ID, then switch to SmartAndPoint" is a later, independent story.

## Dependencies

- [[epics/DDC-002/stories/story-app-skeleton-and-build-pipeline|App skeleton and build pipeline]]
- [[epics/DDC-002/stories/story-cli-clean-an-explicit-item-list|CLI: clean an explicit item list]]
- [[epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision|CLI: shared settings, types and the notify decision]]
- [[epics/DDC-002/stories/story-menu-bar-popover|Menu bar popover]]
- [[epics/DDC-002/stories/story-review-and-clean-window|Review and clean window]]
- [[epics/DDC-002/stories/story-settings-folders-types-notifications|Settings: folders, types, notifications]]
- [[epics/DDC-002/stories/story-background-scan-and-notifications-in-the-app|Background scan and notifications in the app]]

## Attachments

- Design canvas: https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3

---

*Last updated: 2026-10-18*

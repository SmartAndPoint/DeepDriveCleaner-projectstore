---
type: story
id: "story-macos-a-notarized-dmg-for-direct-download"
epic: "DDC-008"
title: "macOS: a notarized DMG for direct download"
status: planned
priority: p3
assignee: "Evgenii Konev"
created: 2026-10-31
updated: 2026-10-31
external_refs: {}
tags: [macos, installer, distribution, dmg, release]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# macOS: a notarized DMG for direct download

| Field | Value |
|---|---|
| **Epic** | [DDC-008](../epic.md) |
| **Status** | planned |
| **Priority** | p3 |
| **Assignee** | Evgenii Konev |

---

## Description

Today the Mac app installs through the Homebrew cask, and the release also carries a zip of the stapled app. The owner wants a native installer file on macOS too: a DMG, the format Mac users expect from a download button (2026-10-31). Windows comes first; this story follows it.

- **The DMG:**
  - `DeepDriveCleaner-<version>.dmg` holds `DeepDriveCleaner.app` and a link to `/Applications`. No custom background: a Finder layout needs a scripted Finder step, which this story does not take on;
  - it is signed with the Developer ID, notarized and stapled by the owner in `make dist`, as the app and the CLI tarball are today.
- **The cask and Sparkle keep the zip.** The DMG is for the website's download button and the GitHub release. This amends one line of [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]] ("A zip, not a DMG: the cask is the install path"). The amendment is dated and names this story.
- **The CLI for DMG users.** The cask links `ddc` onto the `PATH`; a dragged app does not. Decide one of these:
  - a Settings item "Install the command line tool" that links the embedded CLI into `~/.local/bin`, with the same refusal next to a Homebrew install that `install.sh` has;
  - the website pointing DMG users at `install.sh` for the CLI.

## Decomposition

- [ ] `scripts/build-dmg.sh`, called by `make dist` after the app is stapled. Uses `hdiutil` only (no third-party DMG tool). Signs and notarizes the DMG, then staples it.
- [ ] `publish-release.sh` uploads the DMG; `SHA256SUMS` covers it.
- [ ] The ADR amendment, and the CLI decision above, with the critic.
- [ ] The website's macOS download button (`website/`) links the DMG.

## Implementation Plan

## Acceptance Criteria

- [ ] `make dist` produces a DMG that passes `hdiutil verify`, `xcrun stapler validate` and `spctl -a -t open --context context:primary-signature -v`.
- [ ] The DMG is in the release assets and in `SHA256SUMS`.
- [ ] The CLI decision for DMG users is recorded in the ADR amendment, and the chosen route is covered: a test of the Settings item, or the website text pointing at `install.sh`.
- [ ] Manual (owner, the work laptop or another Mac without a development setup), recorded in the owner-checks runbook:
  - the DMG is downloaded by a browser, opened, and the app dragged to Applications;
  - the app starts with only the usual first-launch confirmation;
  - Sparkle later updates that copy.

## Final Summary

## Technical Notes

- The release is a local step on the owner's Mac, which runs unsandboxed because `notarytool` needs the keychain. The DMG adds one more notarization round to `make dist`.

## Dependencies

- None blocking. It comes after the Windows stories by the owner's priority.

## Attachments

-

---

*Last updated: 2026-10-31*

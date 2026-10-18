---
type: story
id: "story-signing-team-parameterise-teamid-then-switch-to-smartandpoint"
epic: "DDC-002"
title: "Signing team: parameterise TEAM_ID, then switch to SmartAndPoint"
status: planned
priority: p3
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-18
external_refs: {}
tags: [signing, release]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Signing team: parameterise TEAM_ID, then switch to SmartAndPoint

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p3 |
| **Assignee** | Evgenii Konev |

---

## Description

macOS names the signer in Login Items & Extensions, the background-activity banner and Gatekeeper dialogs. Today that is a person ("Software from “Evgenii Konev”"), because the Developer ID belongs to the individual team 8MSDJTBW2D. 0.2.0 ships under that team (ADR "Menu bar app: Swift shell over the ddc CLI", §8).

This later, independent story does two things:
- **Parameterise TEAM_ID.** The team id is a constant in two scripts today (`scripts/build-dist.sh:14`, `scripts/publish-release.sh:21`). It becomes one setting that every pipeline reads, the app's included.
- **Switch to SmartAndPoint.** Once the organization team exists, DeepDriveCleaner is signed with its Developer ID. Enrolling is the owner's task outside the repository: D-U-N-S number, enrollment or conversion.

The switch has a one-time cost: the designated requirement includes the team, so every user grants Full Disk Access once more. It also changes the developer name shown in Login Items and in the background-activity banner to the organization's.

## Decomposition

Rough until the plan gate.

- [ ] One setting for the team id (`.env` or a `Makefile` variable), read by `scripts/build-dist.sh`, `scripts/publish-release.sh` and `make app-dist`; the existing team check (`publish-release.sh:132`) and the signing check (`build-dist.sh:57`) use it
- [ ] When the organization team exists: a Developer ID Application certificate, its SHA-1 in `.env` (`DDC_SIGN_IDENTITY`), `notarytool store-credentials` for the new team, the team setting changed
- [ ] Release notes of the first release under the new team say Full Disk Access must be granted once more

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] The team id appears in exactly one place in the repository; with the personal team configured, `make dist` and `make app-dist` produce the same signatures as before
- [ ] After the switch: `codesign -dvv` on the app and the embedded CLI shows `Authority=Developer ID Application: <SmartAndPoint legal name> (<team>)`
- [ ] Manual check after the switch: Login Items & Extensions and the background-activity banner name SmartAndPoint, not "Evgenii Konev"
- [ ] The release notes of that release say Full Disk Access must be granted again

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- The refusal of an asset signed by another team already exists (`publish-release.sh:132`); this story only makes it read the setting.
- GrammarHelper needs the same switch in its own repository.

## Dependencies

- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]
- [[epics/DDC-002/stories/story-distribution-the-020-cask-with-the-app|Distribution: the 0.2.0 cask with the app]]: ships first, under team 8MSDJTBW2D; this story does not block it

## Attachments

-

---

*Last updated: 2026-10-18*

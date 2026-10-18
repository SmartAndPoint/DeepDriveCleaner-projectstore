---
type: story
id: "story-app-skeleton-and-build-pipeline"
epic: "DDC-002"
title: "App skeleton and build pipeline"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-18
external_refs: {design_canvas: "https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3"}
tags: [app, build, signing, spike]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# App skeleton and build pipeline

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

This story builds the shell every other story of the epic builds on. It starts with a spike that checks the ADR's central claims before anything else is built.

The shell:
- an `LSUIElement` app with its own icon, a status item, an empty popover and a right-click menu;
- the CLI embedded in the bundle, and one runner in the app that runs it;
- the scheduler agent's plist in the bundle, a login item and a log file;
- a `make` pipeline that builds, signs, notarizes, staples and zips the app.

It implements §1, §2, §3, §4 and §8 of the ADR "Menu bar app: Swift shell over the ddc CLI", and verifies the attribution claims of §5. GrammarHelper's `Makefile` is the model for the pipeline.

## Decomposition

Rough until the plan gate.

- [ ] **Spike, first** (it blocks every other story of the epic). A minimal app built Debug, signed with the Developer ID, launched through `open` (never from Xcode: the debugger changes attribution), on a fresh user account. It checks:
  - a scan the app spawns reports `full_disk_access: true` with Full Disk Access granted only to the app;
  - an `SMAppService.agent` whose program is `/usr/bin/open -gjb com.smartandpoint.deepdrivecleaner --args --scheduled` launches the quit app with `--scheduled`, and delivers a reopen event to the running app; the scan it leads to also reports `full_disk_access: true`;
  - after the app is quit and replaced by a rebuild with the same bundle id and team, the grant holds and the next tick opens the new app;
  - if `SMAppService` rejects the agent because its program is `/usr/bin/open`, the fallback from the ADR's Open Questions: `BundleProgram: Contents/MacOS/DeepDriveCleaner --tick`, where tick mode only runs the same `open` command and exits
- [ ] `app/DeepDriveCleaner.xcodeproj`: Swift, deployment target macOS 26 (ADR §3), bundle id `com.smartandpoint.deepdrivecleaner`, `LSUIElement`, hardened runtime, no sandbox; Debug builds signed with the same Developer ID as releases, so one TCC identity covers both (GrammarHelper's rule)
- [ ] Icons:
  - an `AppIcon` asset in every macOS size (16 to 512 pt at 1x and 2x), compiled by the build into the bundle's `.icns`;
  - a template glyph for the status item;
  - both drawn as on the canvas: a drive outline with a line and a dot
- [ ] App delegate: `NSStatusItem` with the template glyph, `NSPopover` hosting a SwiftUI placeholder, right-click `NSMenu` (Scan now, Settings…, Quit)
- [ ] One runner for the CLI (ADR §1):
  - locates `Contents/Helpers/deepdrivecleaner` and runs it with `Process`;
  - serialises every run: a scan, a clean, a dry run and `rules --json` never overlap; a scan requested while one runs joins it; a clean requested during a scan starts when the scan ends;
  - reads stdout and stderr concurrently, decodes stdout JSON and forwards stderr progress lines;
  - sends SIGTERM on cancel and maps exit codes 0 / 1 / 2 / 75 to done, error, refused and busy
- [ ] The scheduler agent's plist at `Contents/Library/LaunchAgents/com.smartandpoint.deepdrivecleaner.scheduler.plist`
- [ ] Launch modes as a small state machine (ADR §5):
  - normal: status item, keeps running;
  - `--scheduled`: no status item; quits when its work is done (the work itself belongs to "Background scan and notifications in the app");
  - a reopen or activate event, a further launch (Finder, Spotlight, the login item) or a notification response promotes a `--scheduled` instance to normal: it creates the status item and keeps running, so a scheduled instance never swallows a user launch
- [ ] Login item through `SMAppService.mainApp`; the toggle shows `SMAppService.status` (including `requiresApproval`, with a button that opens Login Items settings), never a stored bool
- [ ] Log file `~/Library/Logs/DeepDriveCleaner/app.log`, next to the CLI's logs
- [ ] `make app`: release-build the CLI, copy it into `Contents/Helpers`, build the app. `make app-dist`: sign the CLI as `com.smartandpoint.deepdrivecleaner.cli`, then the app, notarize, staple, zip, with the `.env` identity, the notary profile and the team of `make dist` (team 8MSDJTBW2D, ADR §8)

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Spike, manual check on a fresh user account with a Debug build signed with the Developer ID and launched through `open`: each check of the spike item passes. The results go into the Final Summary. If any fails, the ADR is revisited before another story of the epic starts
- [ ] `make app-dist` produces `DeepDriveCleaner-<version>.zip`. The app inside passes `codesign --verify --deep --strict`, `spctl --assess --type execute` (accepted, Notarized Developer ID) and `xcrun stapler validate`. The embedded CLI is signed with identifier `com.smartandpoint.deepdrivecleaner.cli` and the same team. The bundle holds the scheduler plist, and `Info.plist` names the compiled icon
- [ ] Manual check: Finder, System Settings → Login Items & Extensions, and a test notification show the app's icon, not a generic one; the status item shows the template glyph in light and dark menu bars
- [ ] Unit test with a fake CLI that counts its runs: two scan requests while a scan runs start one process; a clean requested during a scan starts only after the scan ends; two CLI processes never run at once
- [ ] Unit test: the runner reads a fake CLI that writes 1 MiB to stderr, then 1 MiB to stdout, and finishes without a deadlock
- [ ] Unit test of the launch-mode state machine: a `--scheduled` start has no status item and quits when its work is done; a reopen, an activate, a further launch or a notification response during that work promotes it to normal mode, with the status item, and it does not quit afterwards; a normal start never quits on its own
- [ ] Unit test: the runner decodes a recorded `ddc scan --json` report and treats one with another `schema_version` as unreadable; exit codes 2 and 75 map to refused and busy
- [ ] Manual check: the app has no Dock icon and shows a status item; Quit in the right-click menu exits it; the open-at-login toggle registers and unregisters `SMAppService.mainApp`, and after a change made in System Settings the app shows the system's state

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- The CLI goes in `Contents/Helpers`, not `Contents/MacOS`: on a case-insensitive volume `deepdrivecleaner` and the main executable `DeepDriveCleaner` are the same name.
- Attribution (ADR, Context 1): a child the app spawns is attributed to the app; an app LaunchServices launches is its own responsible process; a CLI started by launchd is its own; a CLI run from Terminal uses Terminal's grant. The spike must therefore run through `open`, and on an account where Terminal has no grant, or it proves nothing.
- GrammarHelper's login-item toggle is a `UserDefaults` bool and drifts from the system's state; do not copy it.
- GrammarHelper's `Assets.xcassets/AppIcon.appiconset` is the model for the icon set; `ASSETCATALOG_COMPILER_APPICON_NAME = AppIcon` makes the build write the `.icns`.
- LaunchServices keeps one instance per bundle id: a user's launch while a `--scheduled` instance runs only reaches that instance, which is why promotion exists.
- The process boundary needs no entitlements; the app is not sandboxed, like GrammarHelper.
- The team id comes from the same constant as `scripts/build-dist.sh`; making it one setting is in "Signing team: parameterise TEAM_ID, then switch to SmartAndPoint".

## Dependencies

- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]
- [[epics/DDC-001/stories/story-homebrew-tap-and-the-v010-release|Homebrew tap and the v0.1.0 release]]

## Attachments

- Design canvas: https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3

---

*Last updated: 2026-10-18*

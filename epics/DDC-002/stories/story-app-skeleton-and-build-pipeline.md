---
type: story
id: "story-app-skeleton-and-build-pipeline"
epic: "DDC-002"
title: "App skeleton and build pipeline"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-20
external_refs: {design_canvas: "https://claude.ai/artifact/W1nniwb2fkg8BX8ReznPN3"}
tags: [app, build, signing]
code_refs: []
specs: []
started_at: "2026-10-18T10:42:53.267Z"
closed_at: null
plan_updated_at: "2026-10-18T10:42:53.267Z"
---

# App skeleton and build pipeline

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

This story builds the shell every other story of the epic builds on. It carries Debug spike tools; the ADR's attribution claims (§5) are checked on the work laptop after 0.2.0 (epic Decisions).

The shell:
- an `LSUIElement` app with its own icon, a status item, an empty popover and a right-click menu;
- the CLI embedded in the bundle, and one runner in the app that runs it;
- the scheduler agent's plist in the bundle, a login item and a log file;
- a `make` pipeline that builds, signs, notarizes, staples and zips the app.

It implements §1, §2, §3, §4 and §8 of the ADR "Menu bar app: Swift shell over the ddc CLI", and ships the tools that check §5's attribution claims. GrammarHelper's `Makefile` is the model for the pipeline.

## Decomposition

Rough until the plan gate.

- [ ] **Spike tools** in Debug builds: Probe Full Disk Access (logs the errno), Register and Unregister Scheduler Agent, Post Test Notification, and the `DebugHoldSeconds` default that keeps a `--scheduled` instance alive. The spike's checks moved to the clean-machine check of "Distribution: the 0.2.0 cask with the app" (owner's decision, 2026-10-18, epic Decisions): the app is finished and dogfooded here first, then checked on the work laptop after the brew install
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

No spec covers this story; the ADR "Menu bar app: Swift shell over the ddc CLI" (§1–§5, §8) is the design. GrammarHelper is the model for the project file, the logger and the icon set; where this plan departs from it, the reason is given.

**Layout.** `app/` at the repo root:
- `app/DeepDriveCleaner.xcodeproj/project.pbxproj`, written by hand from GrammarHelper's:
  - objectVersion 77 with `PBXFileSystemSynchronizedRootGroup`, so a new source file never touches the project;
  - readable fixed object ids and a shared scheme;
  - `SWIFT_VERSION = 6`; `SWIFT_ACTIVE_COMPILATION_CONDITIONS = DEBUG` in Debug, which the `#if DEBUG` spike menu needs;
  - checked with `plutil -lint` and `xcodebuild -list`.

  No xcodegen or tuist is on the machine, and one more tool is not worth it for one target.
- `app/Packages/DDCKit`: a local Swift package, Foundation only, linked through `XCLocalSwiftPackageReference` as GrammarHelper links `CorrectionCore`. All logic the acceptance criteria test lives here and runs under `swift test`, with no test host app carrying the bundle id.
  - `CLIRunner` (an actor): serialises runs, joins a scan requested during a scan, queues a clean behind it; `Process` with stdin `/dev/null`, both pipes read concurrently, stderr split into lines; cancel sends SIGTERM.
  - `RunOutcome`: exit codes 0 / 1 / 2 / 75 → done / error / refused / busy.
  - `ScanReport`: reads `schema_version` first, as the CLI's own store does; anything but 1 is unreadable.
  - `LaunchMode`: a pure `handle(event) -> [Effect]` state machine (normal, `--scheduled`, promotion on reopen, activate, further launch, notification response). A further launch reaches a running instance as a reopen event.
  - `HelperLocator`: builds the `Contents/Helpers/deepdrivecleaner` URL explicitly and refuses it if it equals the main executable. `Bundle.url(forAuxiliaryExecutable:)` looks next to the executable, in `Contents/MacOS`, and on a case-insensitive volume finds the app itself.
- `app/DeepDriveCleaner/`, the app target:
  - `main.swift`: `--tick` runs `open -gjb com.smartandpoint.deepdrivecleaner --args --scheduled` and exits before any `NSApplication` exists (the ADR's Open Questions); otherwise it starts the delegate. A SwiftUI `@main`, as in GrammarHelper, cannot do this. `Info.plist` sets `NSPrincipalClass = NSApplication` and `LSUIElement`.
  - `AppDelegate`: feeds launch arguments, `applicationShouldHandleReopen` and `applicationDidBecomeActive` to `LaunchMode` and applies the effects. A `--scheduled` instance never calls `NSApp.activate` itself, or it would promote itself.
  - `StatusItemController`:
    - the button sends its action on `[.leftMouseUp, .rightMouseUp]`;
    - a left click toggles the `NSPopover`, which hosts a SwiftUI placeholder;
    - a right click sets `statusItem.menu` (Scan now, Settings…, Quit), calls `performClick` and clears the menu in `menuDidClose`.

    GrammarHelper keeps `statusItem.menu` set, which disables the button's action, so a popover would never open.
  - `SettingsWindowController` with the login-item toggle; `LoginItem` wraps `SMAppService.mainApp`, re-reads `status` when the view appears and on activation, and offers `openSystemSettingsLoginItems()` on `.requiresApproval`. Nothing is stored in `UserDefaults`.
  - `AppLog`: GrammarHelper's file-logger pattern, `~/Library/Logs/DeepDriveCleaner/app.log`, directory created first; every launch (with its arguments), reopen, activate and promotion is logged.
  - `DriveGlyph.swift`: the drive outline, line and dot as CoreGraphics paths, the single source of both icons.
  - `Assets.xcassets/AppIcon.appiconset`.
- `app/LaunchAgents/com.smartandpoint.deepdrivecleaner.scheduler.plist`, outside the synchronised folder (inside it the plist would land in Resources), copied by a Copy Files phase to `Contents/Library/LaunchAgents`, where `SMAppService.agent(plistName:)` looks.

**Icons.** No ImageMagick, rsvg or Pillow are assumed. `scripts/icons/main.swift` is compiled with `DriveGlyph.swift` by `swiftc` and writes the ten AppIcon PNGs and `Contents.json` through ImageIO; the PNGs are committed. The app icon fills Apple's squircle on the 824/1024 grid, so Tahoe does not put it on a grey tile. Its design is option B, "Class ring", of the canvas's app icon board (owner, 2026-10-20): the white drive on a dark tile (#1C2026), inside a ring of the four class colours tuned for the dark tile, from the top, clockwise: safe #34B37A, rebuildable #4C86E8, review #E08A2C, user data #8E9299. These values live in `scripts/icons/main.swift` only; they are not `ReportClassStyle`'s. It replaced the first blue tile. The status item draws the glyph at runtime as an 18 pt template image. `ASSETCATALOG_COMPILER_APPICON_NAME = AppIcon` makes the build write the `.icns`.

**Build pipeline.**
- `scripts/build-app.sh [--dist]`:
  1. builds the release CLI;
  2. `xcodebuild -derivedDataPath build/DerivedData CODE_SIGNING_ALLOWED=NO` (all signing in the script: Xcode signing would add `get-task-allow`), then `ditto` to `build/<Config>/` and `xattr -cr`;
  3. copies the CLI to `Contents/Helpers/`, signs it as `com.smartandpoint.deepdrivecleaner.cli`, then signs the app (hardened runtime, timestamp, no `--deep`, no entitlements) and checks the identifier and team of both.

  `--dist` additionally:
  - refuses a dirty tree and a `MARKETING_VERSION` that differs from the Cargo version;
  - zips the app for notarization, notarizes it, and runs `stapler staple` with the retry loop of `scripts/build-dist.sh`, because the ticket can lag;
  - runs the checks of the second criterion;
  - zips the stapled app last, as `dist/DeepDriveCleaner-<version>.zip`.
- `scripts/build-dist.sh` is not touched in this story. `build-app.sh` carries its own small `sign_code` and `retry` helpers. The shared `scripts/lib/release.sh` belongs to "Distribution: the 0.2.0 cask with the app", where `make dist` builds both artefacts and one rerun proves the CLI tarball unchanged. (Note, 2026-10-20: that story deferred the extraction to a later story; it makes `make dist` build both assets, and `make app-dist` is an alias of it.)
- Order of the two targets: `build-dist.sh` starts with `rm -rf dist`, so `make dist` runs first and `make app-dist` after it. `build-app.sh --dist` removes only its own zip. Making `app-dist` part of `dist` is the distribution story's job.
- Makefile:
  - `app` builds Debug and needs only `DDC_SIGN_IDENTITY`; Debug and release share the Developer ID, so one TCC identity covers both.
  - `app-test` runs `swift test` in DDCKit.
  - `app-dist` runs after `check` and `app-test`.
  - `icons` regenerates the icon set.
- `.gitignore`: `/build/`, `xcuserdata/`, `.build/`, `.swiftpm/`. The clean-tree check counts untracked files.

**Tests** (Swift Testing, in DDCKit):
- Fake CLI: a shell script written to a temp directory at test time. It logs `start/end <pid> <args>` and waits on gate files, so the tests need no sleeps and no injected clock.
- Coalescing and ordering: one start for joined scans, the clean after the scan ends, no overlapping intervals.
- 1 MiB of stderr, then 1 MiB of stdout, under a time limit, with the counts checked.
- `LaunchMode`: parameterised over the four promoting events.
- Decoding: a fixture recorded from the synthetic home of `crates/ddc-cli/tests/scan_cli.rs` (no personal paths in a public repo), plus a copy with `schema_version: 2`. The fake CLI exits 2 and 75. A drift test reads `SCHEMA_VERSION` from `crates/ddc-core/src/model.rs` through `#filePath`.

**Order.**
1. The app skeleton, with everything the access checks exercise:
   - the project, `main.swift` with `--tick`, the delegate, the status item with Quit;
   - `AppLog`, and `LaunchMode` with reopen handling and promotion;
   - the `--scheduled` path: run `deepdrivecleaner scan --json`, log `full_disk_access`, quit;
   - the embedded CLI and the agent plist, and `make app`.

   Debug-only additions:
   - a `DebugHoldSeconds` default (`defaults write com.smartandpoint.deepdrivecleaner DebugHoldSeconds 120`) keeps a `--scheduled` instance alive after its work. On a near-empty account the scan ends in under a second, and without the hold there is no running instance to reopen or promote.
   - a Debug menu: Register agent / Unregister agent (logging `SMAppService.status`); Scan now; Probe Full Disk Access; Post test notification (`requestAuthorization`, then one notification, for the third criterion).
   - the probe opens `~/Library/Application Support/com.apple.TCC/TCC.db` read-only, as the CLI's `full_disk_access` does, and logs the errno. `EPERM` is "no access"; `ENOENT` is "no file", and a `false` from the CLI then proves nothing.
2. Checked here without dialogs: `swift test`, an unsigned build, the bundle layout and `Info.plist`, `codesign -dv`. Signing and notarization run in the owner's session, where keychain prompts can appear.
3. Moved (owner's decision, 2026-10-18): the checks below run on the owner's work laptop after `brew install` of 0.2.0, as a criterion of "Distribution: the 0.2.0 cask with the app", with the release app's own status and `app.log` in place of the Debug menu. They are kept here as the list that criterion follows.
   1. The probe and the Scan now item log access; a CLI scan reports `full_disk_access: true`.
   2. Register agent, then three sub-checks, each read from `app.log`. A tick is fired with `launchctl kickstart gui/<uid>/com.smartandpoint.deepdrivecleaner.scheduler`.
      - App quit, tick: a `--scheduled` launch whose scan reports `true`.
      - App launched from Finder (normal mode), tick: a reopen handled as "scan if due" in that instance, no second launch, no promotion.
      - App quit, hold set, tick, then a Finder double-click during the hold: the reopen promotes the `--scheduled` instance, and the status item appears.
   3. Rebuild with the same bundle id and team and replace the app, both ways, recording each:
      - `ditto` over the installed bundle;
      - remove, then copy (what `brew upgrade` does).

      After each, check that the grant holds and the next kickstart opens the new build. GrammarHelper's `CLAUDE.md` records that remove-then-copy orphaned its Accessibility grant. If it does here too, ADR Rationale 1 ("one grant that survives upgrades") does not hold for the cask: the ADR and "Distribution: the 0.2.0 cask with the app" are revised after the release.
   4. Whether `SMAppService` accepts the `open` program is settled on the reference machine during development (the first criterion). It does not when `register()` throws, when the status is neither `.enabled` nor `.requiresApproval`, or when `app.log` shows no launch within 10 s of the kickstart; then the `BundleProgram … --tick` fallback replaces it.
   5. Post test notification: the banner shows the app's icon.
   A failed check sends the ADR back for revision after the release.
4. The runner, the login item, the icons and `app-dist`; the manual criteria last.

**Pitfalls.**
- Never grant or launch a copy Xcode built and ran (⌘R): TCC then stores the ad-hoc requirement, and the Developer ID build reports `full_disk_access: false` with its toggle on.
- Only one copy with this bundle id should exist outside `build/` (`-derivedDataPath build/DerivedData` keeps Xcode's own copy there too), or `open -b` can start a stale one.
- The runner never sets `DDC_DATA_DIR` for the CLI; that would move the CLI's logs.
- The v0.1.0 release is published (asset, tag, tap), so `scripts/publish-release.sh`'s resume guard, which refuses a resume once files outside `packaging/homebrew/` changed, does not bind this work.

**Proposed `code_refs`.**
- Story: `app/`, `scripts/build-app.sh`, `scripts/icons/`, `Makefile`, `.gitignore`.
- Epic DDC-002: `app/`, `scripts/build-app.sh`, `scripts/icons/`.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Manual check on the reference machine with the Debug build: after Register Scheduler Agent, `SMAppService` reports `.enabled` or `.requiresApproval`, and `launchctl kickstart gui/<uid>/com.smartandpoint.deepdrivecleaner.scheduler` with the app quit logs a `--scheduled` launch in `app.log`; this settles the `open` program against the `--tick` fallback. Probe Full Disk Access logs its errno and Post Test Notification posts. (The attribution checks moved to "Distribution: the 0.2.0 cask with the app", owner's decision 2026-10-18.)
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
- Attribution (ADR, Context 1): a child the app spawns is attributed to the app; an app LaunchServices launches is its own responsible process; a CLI started by launchd is its own; a CLI run from Terminal uses Terminal's grant. The clean check must therefore start the app through LaunchServices (Finder, `open`, the scheduler), on a Mac where Terminal has no grant, or it proves nothing.
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

*Last updated: 2026-10-20*

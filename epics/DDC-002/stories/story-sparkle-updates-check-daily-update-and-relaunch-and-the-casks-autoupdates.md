---
type: story
id: "story-sparkle-updates-check-daily-update-and-relaunch-and-the-casks-autoupdates"
epic: "DDC-002"
title: "Sparkle updates: check daily, Update and Relaunch, and the cask's auto_updates"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-29
updated: 2026-10-31
external_refs: {}
tags: [app, updates, sparkle, homebrew, distribution, signing, network, adr-amendment]
code_refs: ["app/DeepDriveCleaner.xcodeproj/project.pbxproj", "app/DeepDriveCleaner.xcodeproj/project.xcworkspace/xcshareddata/swiftpm/Package.resolved", "app/DeepDriveCleaner/AppDelegate.swift", "app/DeepDriveCleaner/GeneralSettingsView.swift", "app/DeepDriveCleaner/Info.plist", "app/DeepDriveCleaner/PopoverView.swift", "app/DeepDriveCleaner/SettingsWindowController.swift", "app/DeepDriveCleaner/SparkleUpdater.swift", "app/DeepDriveCleaner/StatusItemController.swift", "app/DeepDriveCleaner/UpdateLineView.swift", "app/DeepDriveCleaner/UpdaterSettings.swift", "app/Packages/DDCKit/Sources/DDCKit/AppPreferences.swift", "app/Packages/DDCKit/Sources/DDCKit/CleanRun.swift", "app/Packages/DDCKit/Sources/DDCKit/DiskFreeRun.swift", "app/Packages/DDCKit/Sources/DDCKit/DiskModel.swift", "app/Packages/DDCKit/Sources/DDCKit/LaunchMode.swift", "app/Packages/DDCKit/Sources/DDCKit/PopoverModel.swift", "app/Packages/DDCKit/Sources/DDCKit/ReviewModel.swift", "app/Packages/DDCKit/Sources/DDCKit/RunnerActivity.swift", "app/Packages/DDCKit/Sources/DDCKit/ScheduledWork.swift", "app/Packages/DDCKit/Sources/DDCKit/UpdateFlow.swift", "app/Packages/DDCKit/Sources/DDCKit/UpdateModel.swift", "app/Packages/DDCKit/Tests/DDCKitTests/CleanRunTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/DiskFreeTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/LaunchModeTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/PopoverModelTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ReviewSupport.swift", "app/Packages/DDCKit/Tests/DDCKitTests/RunnerActivityTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ScheduledScanTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/UpdateFlowTests.swift", "app/Packages/DDCKit/Tests/DDCKitTests/UpdateModelTests.swift", "Makefile", "packaging/homebrew/deepdrivecleaner.rb.tmpl", "packaging/sparkle/appcast.xml.tmpl", "README.md", "scripts/build-app.sh", "scripts/build-dist.sh", "scripts/check-build-number.sh", "scripts/check-signatures.sh", "scripts/debug-feed.sh", "scripts/publish-release.sh", "scripts/render-appcast.sh", "scripts/tests/test_appcast.py", "scripts/tests/test_build_number.py", "scripts/tests/test_cask.py", "scripts/tests/test_debug_feed.py", "scripts/tests/test_signing_check.py"]
specs: []
started_at: "2026-10-29T15:26:21.323Z"
closed_at: "2026-10-30T07:25:02.515Z"
plan_updated_at: "2026-10-29T15:26:21.323Z"
---

# Sparkle updates: check daily, Update and Relaunch, and the cask's auto_updates

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | done |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner, 2026-10-29: the app checks for a new release, shows it in the popover and in Settings, and installs it when the person presses **Update and Relaunch**. Updates go through Sparkle 2 and must work next to the Homebrew cask. The decision is the ADR [[adr/updates-through-sparkle-alongside-the-homebrew-cask|Updates through Sparkle, alongside the Homebrew cask]]; the analysis is the research note [[research/update-check-and-update-in-place|Update check and update in place]]. The ADR amends ADR [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]] (§5, §8) and reverses its "no Sparkle" line.

The owner's rules:

- The check is on by default, at most once a day; a Settings switch turns it off.
- Install only on **Update and Relaunch**.
- Never during a scan or a clean, never in a `--tick` or `--scheduled` launch.
- A team change is never installed this way.

The story has two parts that can ship in one release: the release tooling (key, signing, appcast, cask) and the app integration (driver, gating, UI). If the plan finds them too large together, it splits the tooling into its own story in this epic; that is the only split.

## Decomposition

**Release tooling**

- [x] The owner runs `generate_keys` once and exports an offline backup (`generate_keys -x`). The public key goes into Info.plist as `SUPublicEDKey`. — evidence: `SUPublicEDKey` in `app/DeepDriveCleaner/Info.plist:39` (4784317); the offline backup is the owner's check
- [x] Add Sparkle 2 to the app target as a Swift package, pinned to an exact version. Info.plist: `SUFeedURL`, `SUPublicEDKey`, `SUEnableAutomaticChecks` YES, `SUScheduledCheckInterval` 86400, `SUAutomaticallyUpdate` NO, `SUAllowsAutomaticUpdates` NO, `SUVerifyUpdateBeforeExtraction` NO (the default, kept explicit). — evidence: caa8ce0, c1ae725; Sparkle 2.10.1 exact in `Package.resolved`; Info.plist keys
- [x] `build-app.sh`: delete the framework's XPC services; sign `Autoupdate`, `Updater.app`, then `Sparkle.framework` with the Developer ID, hardened runtime and timestamp, keeping Sparkle's identifiers; then the CLI and the app as today. — evidence: c1ae725; signed `make app`: "5 Mach-O files signed by team … with the hardened runtime"
- [x] `build-app.sh`: a check that every Mach-O file in the bundle carries the expected team and the hardened runtime flag; `make dist` fails otherwise. Zip with `ditto -c -k --sequesterRsrc --keepParent`. — evidence: 97f595b, `scripts/check-signatures.sh`, `scripts/tests/test_signing_check.py`
- [x] `build-app.sh --dist`: refuse unless `CURRENT_PROJECT_VERSION` is greater than the one in `project.pbxproj` at the last release tag. — evidence: 0238e11, `scripts/check-build-number.sh` called from `build-app.sh:65` and `build-dist.sh:45`
- [x] `packaging/sparkle/appcast.xml.tmpl` (the root declares the `sparkle` and `ddc` namespaces) and a render step: a script `publish-release.sh` calls, with the `sign_update` path injectable for the test. — evidence: b9cb548, `scripts/render-appcast.sh`, `scripts/tests/test_appcast.py`
- [x] `publish-release.sh`, before the release exists: run `sign_update`, render the appcast, check it with `xmllint --noout`, refuse a `CFBundleVersion` that is not greater than the previous appcast's, refuse a team change unless `--team-change` (which renders `informationalUpdate`). Pass `appcast.xml` to `gh release create` as a third asset. Upload a missing appcast on the resume and "tap current" paths. `--dry-run` prints the rendered appcast. — evidence: ffbddd4, 9f95dff (team from the zip)
- [x] `publish-release.sh`, after the tap write: fetch the live `releases/latest/download/appcast.xml`, check it with `xmllint`, and compare its `sparkle:version` and `edSignature` with the rendered file; fail with a message on a mismatch. This runs only in the release script. — evidence: ffbddd4; `check_live_feed` retries 6 times, 10 s apart (9f95dff)
- [x] Cask template: `auto_updates true`; `zap trash:` adds `~/Library/Caches/com.smartandpoint.deepdrivecleaner` and `~/Library/HTTPStorages/com.smartandpoint.deepdrivecleaner` in alphabetical order (paths confirmed on a built app). — evidence: ffbddd4, `scripts/tests/test_cask.py`
- [x] An ops note for the first real pair of Sparkle releases: the first ships the updater and is reached through `brew upgrade` or by hand; the second is the first update through Sparkle, checked on a Homebrew copy and a hand-dragged copy; what to do if the live-feed check fails. — evidence: vault runbook "First Sparkle releases: shipping the updater, then proving it" (3d7985b)

**App integration**

- [x] DDCKit: `UpdateFlow`, the pure state machine of the ADR's §4. "Busy" comes from `CLIRunner`'s state only; there is no data-dir lock probe. — evidence: 8f4df6d, 9f95dff; busy from `RunnerActivity` (430b989)
- [x] DDCKit: `ScheduledWork.tick` (`ScheduledWork.swift:85`) starts no scan while an install waits. — evidence: 2e2145e
- [x] DDCKit: `LaunchMode` reports when the updater may start (the first `showStatusItem`), so the gating is tested where `LaunchMode` is. — evidence: 2e2145e, `LaunchModeTests`
- [x] App: `SparkleUpdater.swift`, the `SPUUserDriver` and `SPUUpdaterDelegate` adapter. Found → record and Dismiss; press → user-initiated check, then Install with no second press; information only → never Install; `shouldPostponeRelaunchForUpdate` holds while the runner has work and resumes when it is idle. `userAgentString` = `DeepDriveCleaner`. — evidence: df5017a, 9f95dff
- [x] App: the Debug-only feed override, under `#if DEBUG`: `feedURLStringForUpdater` returns `http://127.0.0.1:<port>/appcast.xml`, with an App Transport Security exception for that host in the Debug build only, and a hand-rendered appcast and test zip to serve. Debug builds never read the production feed. — evidence: df5017a, `scripts/debug-feed.sh`, `scripts/tests/test_debug_feed.py`
- [x] App: the popover line ("Version X is available", notes link, **Update and Relaunch**, progress, "Waiting for the scan to finish", errors), Scan Now and the clean and free buttons disabled while an install waits, and the Settings (General) rows: the switch bound to `automaticallyChecksForUpdates`, the last check time, **Check Now**, and "Updates are unavailable in this build" when `SPUUpdater.start()` throws. — evidence: 70a7838; Check Now enabled only at rest (9f95dff)
- [x] `AppDelegate`: create the updater at launch, start it on the first `showStatusItem` effect, on the main thread, and handle a thrown start as above. — evidence: 2e2145e, df5017a
- [x] README: a "Network" section (GitHub sees the IP address, the time, the `User-Agent` `DeepDriveCleaner` with no version, and the system language through `Accept-Language`; the release's download counter for `appcast.xml` gives the maintainers a total count only; system profiling is off; one request at the first launch of the first Sparkle release, then at most daily; the Settings switch is the opt-out). The Install text says the app updates itself and that CLI-only Homebrew users run `brew upgrade --greedy deepdrivecleaner`; the Full Disk Access text mentions Sparkle updates next to `brew upgrade`; rollback is `brew reinstall` or an older release page. — evidence: 4478e83, b8a4d38 (plain `brew upgrade` upgrades a cask that is behind; `--greedy` only with `HOMEBREW_NO_UPGRADE_AUTO_UPDATES_CASKS`)

## Implementation Plan

One story, in about twelve commits. The planner (opus) placed it; this is the route. The ADR "Updates through Sparkle, alongside the Homebrew cask" is the normative "how".

**Order matters:** the cask's `auto_updates true` and the publish-release integration land last (C11), after the updater is in the app. A cask that says the app updates itself must never ship in a build that cannot. No release is cut between C4 and C11.

1. **C1 Appcast render and the script-test harness.** `packaging/sparkle/appcast.xml.tmpl`, `scripts/render-appcast.sh` (all inputs as flags, `sign_update` from `SPARKLE_SIGN_UPDATE`, the build-number and team guards, `--team-change` renders `informationalUpdate`, ends with `xmllint --noout`), `scripts/tests/test_appcast.py` with a fake `sign_update`, a `script-test` target in `make gate`. Reuses publish-release's placeholder pattern and the `bench/test_*.py` style. xpath uses `local-name()`.
2. **C2 Per-binary signing check.** `scripts/check-signatures.sh <app> <team>`: every Mach-O has the team and the runtime flag; called by `build-app.sh` after `--verify --deep --strict`. The fail case (ad-hoc fixture) runs in `make app-test`.
3. **C3 Build number and the zip.** `scripts/check-build-number.sh` compares `CURRENT_PROJECT_VERSION` with the last `v*` tag's (after `git fetch --tags`); called by `build-dist.sh` before notarizing and by `build-app.sh --dist`. The zip gets `--sequesterRsrc`; check the listing for `__MACOSX/` on the first build.
4. **C4 `RunnerActivity` (DDCKit).** A `@MainActor @Observable` decorator over `CLIRunning` counting runs in flight per kind, with `busy`, a reason and `onIdle`. AppDelegate wraps the one `CLIRunner`. The busy answer is synchronous, as Sparkle's postpone question needs.
5. **C5 `UpdateFlow` (DDCKit).** A pure state machine, shaped like `LaunchMode`; replies only Dismiss and Install, never Skip.
6. **C6 `UpdateModel` (DDCKit).** Holds the flow and the stored postpone handler, invokes it once on idle, drops it on quit; the found version, short version, release URL and feed URL in `AppPreferences` (a record from another feed is ignored: Debug and Release share defaults).
7. **C7 Launch and job gates (DDCKit).** `LaunchMode.Effect.startUpdater` after `.showStatusItem`; `ScheduledWork` gets `updateWaiting` and `TickResult.updateWaiting`; the scan, clean, free and uninstall buttons are gated while an install waits.
8. **C8 Sparkle in the project.** Remote package pinned exact (2.10.1) by hand in the pbxproj, `Package.resolved` committed, `-disableAutomaticPackageResolution`; Info.plist keys from ADR §1; `build-app.sh` removes the XPC services (directory and link) and signs Autoupdate, Updater.app and the framework with Sparkle's own identifiers, in Debug too; `--dist` refuses a missing `SUPublicEDKey`.
9. **C9 The adapter.** `SparkleUpdater.swift`: `@MainActor` `SPUUserDriver` + `SPUUpdaterDelegate`, completion-handler forms only, `userAgentString = "DeepDriveCleaner"`, `#if DEBUG` feed on 127.0.0.1 with the port from `DebugFeedPort`; `start()` only on `.startUpdater`, a throw becomes "unavailable"; `.appQuits` at the top of `applicationShouldTerminate`. `scripts/debug-feed.sh` renders a local feed. `--dist` asserts no `127.0.0.1`, no `NSAppTransportSecurity`, the production `SUFeedURL`.
10. **C10 UI.** `UpdateLineView` in the popover; `UpdaterSettings` (switch bound to `automaticallyChecksForUpdates`, last check, Check Now, the unavailable text) in General settings.
11. **C11 Release and cask.** publish-release: `sign_update` from the resolved SPM artifact, `generate_keys -p` must match the zip's key, the previous appcast read before the release, `appcast.xml` as a third asset of `gh release create`, uploaded when missing on resume, the live feed checked after the tap write. Cask: `auto_updates true`, two `zap` paths in order; `scripts/tests/test_cask.py`.
12. **C12 README.** "Network", the update text under Install and Full Disk Access, rollback. The ops note for the first real pair of releases.

**Checks per commit:** `make gate-fast` covers Rust only; the meaningful checks here are `make app-test`, the new script tests, and an unsigned compile of the app (`CODE_SIGNING_ALLOWED=NO`).

**Waits on the owner:** `generate_keys` (the key in the login keychain and its backup) before the first `make dist` after C8. Until then a release stops at the missing key.

## Acceptance Criteria

- [x] Test (app): `UpdateFlow`: a found item becomes "available" and is replied Dismiss; a press while idle leads to Install on the next found reply and on ready-to-install, with no second press; a press while the runner has work installs nothing until it is idle; an information-only item is never answered Install; an error returns to "available" with the release link; a start that throws gives "unavailable". — evidence: `app/Packages/DDCKit/Tests/DDCKitTests/UpdateFlowTests.swift`: aFoundItemBecomesAvailableAndIsDismissed, onePressInstallsOnTheFoundReplyAndOnReadyToInstall, aPressWhileTheRunnerHasWorkInstallsNothingUntilItIsIdle, anInformationOnlyItemIsNeverInstalled, anErrorAfterAPressReturnsToAvailableWithTheReleaseLink, aStartThatThrowsGivesUnavailable
- [x] Test (app): postponing: with the runner busy, the postpone decision is "hold"; when the runner becomes idle, the stored handler is called exactly once. — evidence: UpdateFlowTests.theHeldRelaunchResumesOnceWhenTheRunnerIsIdle, UpdateModelTests.withTheRunnerBusyTheRelaunchIsHeldAndResumesOnce
- [x] Test (app): "waiting for idle" followed by "app quits" ends in "installing on quit": no relaunch is asked for and the handler is not called twice. — evidence: UpdateFlowTests.aQuitWhileWaitingEndsInInstallingOnQuit, UpdateModelTests.aQuitWhileWaitingDropsTheHandler
- [x] Test (app): `ScheduledWork.tick` starts no scan while an install waits, and starts one again after. — evidence: ScheduledScanTests "a tick starts no scan while an update's install waits, and scans again after"
- [x] Test (app): launch-mode gating: a `--scheduled` launch that finishes its work never reaches "start updater"; a `--scheduled` launch promoted by reopen, activate or a notification response reaches it once; a normal launch reaches it once. — evidence: LaunchModeTests: "a --scheduled launch that finishes its work never starts the updater", "… promoted by a user's event starts the updater once", "a normal launch starts the updater once, after the status item"
- [x] Test (app): the found version kept in `UserDefaults` is cleared when the running `CFBundleVersion` reaches or exceeds it. — evidence: UpdateModelTests.theKeptUpdateIsClearedWhenTheRunningBuildReachesIt, theKeptUpdateIsComparedAsANumber
- [x] Test (app): a Release build never reads the Debug feed override (the override is compiled out, and the feed is `SUFeedURL`). — evidence: the override is under `#if DEBUG` (`SparkleUpdater.swift:58-63`); `scripts/build-app.sh:162-170` fails a Release binary that names 127.0.0.1 and a Debug one that does not; Release and Debug xcodebuild green 2026-10-30
- [x] Test (script): the appcast render, with a fake `sign_update` that prints a fixed `sparkle:edSignature="…" length="…"`, produces XML that passes `xmllint --noout`, in which `xmllint --xpath` finds `enclosure/@sparkle:edSignature`, with the version, short version, `minimumSystemVersion` `26.0.0`, the versioned enclosure URL, length, the full release notes link and the `ddc:team` element; it refuses a `CFBundleVersion` not greater than the previous appcast's; it refuses a team change without `--team-change` and renders `informationalUpdate` with it. No network. — evidence: `scripts/tests/test_appcast.py`: test_renders_one_signed_item_that_xmllint_accepts, test_refuses_a_version_not_greater_than_the_previous_one, test_refuses_a_team_change_without_the_flag, test_a_team_change_renders_information_only_with_no_enclosure
- [x] Test (build): the per-binary signing check fails on a fixture bundle signed ad hoc (`codesign -s -`) inside `Sparkle.framework`; this runs anywhere, in `make app-test`. The pass case runs on a Developer ID build in `make dist`. — evidence: `scripts/tests/test_signing_check.py` test_fails_on_one_file_signed_ad_hoc_inside_the_framework; pass case on the Developer ID `make app` 2026-10-30
- [x] Test (build): `build-app.sh --dist` refuses a `CURRENT_PROJECT_VERSION` not greater than the last tag's. — evidence: `scripts/tests/test_build_number.py` test_refuses_a_build_number_equal_to_or_below_the_last_tag
- [x] Test (build): the rendered cask contains `auto_updates true` and the two `zap` paths in alphabetical order, and passes `ruby -c` and `brew style` (the existing publish checks). — evidence: `scripts/tests/test_cask.py`: test_zap_trashes_sparkles_cache_and_http_storage_in_alphabetical_order, test_the_rendered_cask_is_valid_ruby, test_the_rendered_cask_passes_brew_style (with `DDC_TEST_BREW_STYLE=1`)
- Manual checks (5) moved to [Owner checks: manual acceptance across stories](../../../ops/owner-checks-manual-acceptance-across-stories.md) on 2026-10-30, by the owner's decision.

## Final Summary

**What changed.** The app updates itself through Sparkle 2.10.1 next to the Homebrew cask. A custom `SPUUserDriver` drives the popover's update line through the pure `UpdateFlow` state machine in DDCKit: a found update is recorded and dismissed, **Update and Relaunch** checks and installs with one press, an information-only item (a team change) never installs, and Sparkle's postpone question holds the relaunch while the runner has work, from any path into it. Scheduled scans and the clean and free buttons wait while an install runs. Settings → General has the daily-check switch, the last check time and **Check Now** (enabled only when the line is at rest). The release builds sign Sparkle's binaries, check every Mach-O file for the team and the hardened runtime, refuse a build number that does not grow, render and sign `appcast.xml`, attach it to the release, and read the live feed back. The cask says `auto_updates true` and zaps Sparkle's cache and HTTP storage. The README has a Network section.

**Why.** The owner wanted updates in the app that still work with Homebrew. A plain `brew upgrade` upgrades the cask while the app is behind the tap and leaves it alone after a Sparkle update.

**Tests executed (2026-10-30, at b8a4d38; on main through the merge e3f7eb9).** `swift test` in DDCKit: 378 tests in 41 suites passed. Script tests: 31 run, OK, 2 skipped (`brew style` is opt-in). `xcodebuild` Debug and Release unsigned: no errors. Signed Debug `make app`: valid, "5 Mach-O files signed by team … with the hardened runtime". The reviewer's findings F1–F3, N1, N2, N4, N5 and O1 were fixed in 9f95dff; the push ran `make gate`. The merge e3f7eb9 fits the appcast into the tag-first release flow: `appcast.xml` is a release asset beside `SHA256SUMS`, not in it, and a pre-release carries none.

**Risks and follow-ups.**
- The update path is first exercised by the release after 0.8.0; the runbook "First Sparkle releases: shipping the updater, then proving it" covers it.
- The KVO of `canCheckForUpdates` and the Debug-feed end-to-end run are the owner's manual checks.
- A postpone question after a quit is ignored on purpose: Sparkle finishes that install without a relaunch.
- `CURRENT_PROJECT_VERSION` must be raised in every release commit.
- Released in 0.8.0 (tag at 356564c). Its first `make dist` stopped at the zip: `ditto --sequesterRsrc` packed the `com.apple.provenance` attribute into `__MACOSX/`. 356564c zips with `--norsrc --noextattr` instead, and the check refuses `._` entries too.

**Amendment, 2026-10-31 (the owner).** A Check Now that found nothing left Settings → General unchanged, so the press looked like it did nothing. Settings → General now has a "Current version" row above the daily check: the running version and build, and under it what the last check found ("Up to date", "<version> is available", "Checking…", "Not checked yet", "The last check failed"). `UpdateModel.versionStatus(lastCheck:)`; tests `UpdateModelTests.theVersionRowSaysWhatTheLastCheckFound`, `.theVersionRowNamesAKeptUpdateAndAFailedCheck`. PR #26 (d4d674f), shipped in 0.12.1.

## Technical Notes

- **Order in `publish-release.sh`.** Sign and render the appcast, then `gh release create` with the zip, the tarball and `appcast.xml`, then digests, then the cask to the tap, then the live-feed check. The release becomes "latest" with its feed in place, so the feed URL never returns 404 during a publish. Until the tap is written, the cask is one version behind; a `brew upgrade --greedy` in those minutes reinstalls the previous version, and the next check offers the update again.
- **`sign_update` and the keychain.** Like `notarytool`, `sign_update` needs the login keychain, so the release runs unsandboxed. The tool comes from the SPM checkout (`…/SourcePackages/artifacts/sparkle/Sparkle/bin/sign_update`); the script finds it under the build's DerivedData and fails with a clear message if it is missing.
- **Team guard.** Sparkle accepts a changed Apple certificate when the EdDSA signature is valid, so it would install a team change. The guard is in the release script: it reads `TeamIdentifier` from the app in the zip and compares it with the previous appcast's `ddc:team`. A team change and an EdDSA key change never ship in the same release.
- **No lock probe.** Rust's `try_lock` never waits (`crates/ddc-core/src/store.rs:41-54`), so an app probe of `<data-dir>/lock` could make a Terminal `ddc` exit 75 for no reason. A Terminal `ddc` is safe without it: `SUPlainInstaller` moves the old bundle aside and a running process keeps its inode.
- **Quit while waiting.** Accepted: Sparkle's installer finishes after the host quits (`finishInstallationAfterHostTermination`), so the update installs without a relaunch. An orphaned scan child runs on from the old inode.
- **First Sparkle release.** 0.7.0 and earlier have no Sparkle; people reach the first Sparkle release through `brew upgrade` or by hand. The story's end-to-end check runs on a Debug build against a local feed; the first real update through Sparkle is the ops note's.
- **Question for the plan:** the exact `zap` paths on a built app.
- **Not in scope:** `ddc update`, a CLI self-update, delta updates, a beta channel, the website's privacy text.

## Dependencies

- ADR [[adr/updates-through-sparkle-alongside-the-homebrew-cask|Updates through Sparkle, alongside the Homebrew cask]] accepted at this story's work-start gate.
- The owner's EdDSA key (manual, before the first release with Sparkle).
- The ops note for the first real pair of Sparkle releases (written in this story).

## Attachments

-

---

*Last updated: 2026-10-31*

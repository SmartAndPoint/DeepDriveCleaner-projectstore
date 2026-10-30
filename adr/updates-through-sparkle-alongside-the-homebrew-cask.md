---
type: adr
id: "updates-through-sparkle-alongside-the-homebrew-cask"
title: "Updates through Sparkle, alongside the Homebrew cask"
status: accepted
date: 2026-10-29
authors: ["Evgenii Konev"]
tags: [app, updates, sparkle, homebrew, distribution, signing, network, privacy]
external_refs: {}
supersedes: null
superseded_by: null
review_status: approved
reviewed_at: 2026-10-30
---

# Updates through Sparkle, alongside the Homebrew cask

| Field | Value |
|---|---|
| **Status** | proposed |
| **Date** | 2026-10-29 |
| **Authors** | Evgenii Konev |

---

## Context

The research note [[research/update-check-and-update-in-place|Update check and update in place]] compares the routes for an in-app update: (a1) a detached `brew upgrade` for Homebrew installs, (a2) our own bundle swap, (b) Sparkle with `auto_updates true` in the cask, (c) notify only. The owner first picked (a1) on 2026-10-29, then changed it the same day: updates go through **Sparkle 2**, and they must work correctly next to Homebrew.

The owner's decisions that stand from the first answer:

- The check is on by default, at most once a day. Settings can switch it off.
- A found update shows in the popover and in Settings.
- Install happens only when the person presses **Update and Relaunch**, never on its own.
- No update while a scan or a clean runs, and never in the `--scheduled` launch mode.
- A change of the signing team is never installed this way.

What the code has today (read 2026-10-29):

- `app/DeepDriveCleaner/Info.plist`: `CFBundleShortVersionString` is `$(MARKETING_VERSION)` (0.7.0), `CFBundleVersion` is `$(CURRENT_PROJECT_VERSION)` (12), `LSMinimumSystemVersion` is `$(MACOSX_DEPLOYMENT_TARGET)`, `LSUIElement` is true. No sandbox, no entitlements file, hardened runtime on. The only package reference is the local `Packages/DDCKit`.
- `scripts/build-app.sh` builds with `CODE_SIGNING_ALLOWED=NO`, copies the CLI into `Contents/Helpers`, signs inside-out without `--deep` (the CLI, then the app), checks each signature's identifier and team, notarizes, staples, and zips with `ditto -c -k --keepParent` to `dist/DeepDriveCleaner-<v>.zip`.
- `scripts/publish-release.sh` creates the GitHub release with the zip and the CLI tarball, checks the uploaded digests, renders the cask from `packaging/homebrew/deepdrivecleaner.rb.tmpl` and writes it to the tap through the contents API. It has a `--dry-run` and a resume path for an existing release.
- The cask: `app`, two `binary` links into `Contents/Helpers/deepdrivecleaner` (one named `ddc`), `uninstall quit:` with the bundle id, `zap` of three launchd labels and the data, logs and preferences. No `auto_updates`.
- `main.swift`: a `--tick` launch runs `/usr/bin/open` on the bundle with `--args --scheduled` and exits before `NSApplication` exists. `LaunchMode` (DDCKit) turns a `--scheduled` launch into `startScheduledWork`, and promotes it to normal (`showStatusItem`) on reopen, activate or a notification response.
- The data-dir lock (`<data-dir>/lock`, `store.rs`) is a `flock`: scans hold it shared, `clean` and `disk free` exclusive, and `try_lock` never waits.
- The app re-registers its `SMAppService` agent at every launch (ADR [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]], §5). Full Disk Access is keyed by the bundle id and a designated requirement that includes the team (§8, "Team").
- The app makes no network request of its own today.

What Sparkle 2 gives, from its documentation (sparkle-project.org, read 2026-10-29):

- `SUFeedURL`, `SUPublicEDKey` (EdDSA), `SUEnableAutomaticChecks` (unset means "ask at the second launch"), `SUScheduledCheckInterval` (default 86400, minimum one hour), `SUAutomaticallyUpdate` (default NO), `SUAllowsAutomaticUpdates`, `SUEnableSystemProfiling` (default NO). The installer and downloader XPC services are only for sandboxed apps.
- `SPUUpdater.automaticallyChecksForUpdates` is stored in the host app's user defaults and is KVO-compliant. `userAgentString` is settable; the default is `<name>/<version> Sparkle/<version>`.
- `SPUUpdaterDelegate`: `updater(_:shouldPostponeRelaunchForUpdate:untilInvoking:)` lets the app hold the install and relaunch and resume it later.
- A custom user driver (`SPUUserDriver`) replaces Sparkle's windows. For apps without a Dock icon the standard driver offers "gentle reminders" (2.2+).
- Keys: `generate_keys` stores the private key in the login keychain and prints the public key; `-x` exports it, `-f` imports it. An update may change the Apple code signing certificate or the EdDSA key, never both. A lost EdDSA key can be replaced in a release signed with the same Developer ID.
- `sign_update <archive>` prints `sparkle:edSignature="…" length="…"`. The tools ship in the SPM artifact under `artifacts/sparkle/Sparkle/bin/`.
- A non-sandboxed app may delete the framework's XPC services and re-sign it; manual signing goes inside-out: `Autoupdate`, `Updater.app`, then the framework.
- The downloader (`SPUDownloader`) sets no redirect callback, so it follows `NSURLSession`'s default redirects. Only the first URL's scheme is checked (http or https). App Transport Security, with no exceptions in this app, blocks a plain-http hop.

What Homebrew says (Cask Cookbook): `auto_updates true` is for an app that has its own "Check for Updates" which downloads and installs. `brew upgrade` then skips the cask unless `--greedy` or `--greedy-auto-updates` is given. `uninstall quit:` runs on uninstall, upgrade and reinstall. The coordinator noted that homebrew/cask will expect `auto_updates` for self-updating apps from 2026-11-16. The Acceptable Casks page did not state that on 2026-10-29, so this ADR treats it as unverified; the decision does not depend on it.

## Decision

### 1. Sparkle 2, in the app only

The app target adds Sparkle 2 as a Swift package, pinned to an exact 2.x version (the plan picks the newest at implementation). DDCKit does not link it: a thin adapter in the app target maps Sparkle's calls into a DDCKit state machine, `UpdateFlow`, which is tested without Sparkle (§4). The CLI does not change.

Info.plist gains:

| Key | Value | Why |
|---|---|---|
| `SUFeedURL` | `https://github.com/SmartAndPoint/DeepDriveCleaner/releases/latest/download/appcast.xml` | GitHub's stable URL for the latest release's asset; no API, no rate limit |
| `SUPublicEDKey` | the owner's public key (§7) | every archive is EdDSA signed |
| `SUEnableAutomaticChecks` | YES | the owner's default; Sparkle does not ask at the second launch |
| `SUScheduledCheckInterval` | 86400 | at most once a day |
| `SUAutomaticallyUpdate` | NO | install only on the button |
| `SUAllowsAutomaticUpdates` | NO | Sparkle never offers silent installs |

| `SUVerifyUpdateBeforeExtraction` | NO (the default) | rotation via the same zip artifact outweighs pre-extraction verification for an HTTPS-only GitHub feed; with YES, a key rotation would need a Developer ID signed disk image instead of the zip |

`SUEnableSystemProfiling`, `SUEnableInstallerLauncherService` and `SUEnableDownloaderService` are not set.

**Development feed.** In Debug builds only (`#if DEBUG`), the delegate's `feedURLStringForUpdater` returns a local feed, `http://127.0.0.1:<port>/appcast.xml`, served from a hand-rendered appcast and a test zip, with an App Transport Security exception for that host in the Debug build only. Debug builds never read the production feed. Release builds do not compile the override, and a test checks that they never read it.

The feed URL redirects twice (to the versioned release asset, then to GitHub's asset host), all HTTPS. Sparkle follows both. A prerelease is never "latest", so it is never offered.

### 2. A custom user driver, not Sparkle's windows

The app implements `SPUUserDriver`. Sparkle opens no window while the app runs. After the app quits for the install, Sparkle's `Updater.app` may show its progress window, or macOS an authorization prompt (§9).

- **Found.** `showUpdateFound` records the version, the short version and the release page, and replies **Dismiss** at once. With automatic updates off, Dismiss downloads, installs and persists nothing, and the next check offers the same item again. Sparkle's session ends, so the next daily check still runs and sees a newer release. The app keeps the found version in `UserDefaults` itself and clears it when the running `CFBundleVersion` reaches or exceeds it, so the line survives a restart.
- **Shown.** The popover gets one line, "Version X is available", with a link to the release notes and **Update and Relaunch**. Settings (General) shows the same, plus the last check time, **Check Now**, and the switch of §5.
- **Pressed.** The press starts a user-initiated check (`checkForUpdates()`). The driver remembers the press, so when Sparkle reports the same item it replies **Install** at once. Download and extraction show their progress in the popover line, and `showReadyToInstallAndRelaunch` is answered **Install** with no second press. Errors show in the line, with the release page as the manual route.
- **Information only.** An item that is information only (`informationalUpdate`, §6) is never installed: the line says "Version X needs a manual install" and links to the release page. **Update and Relaunch** is not shown.
- **Background errors** (no network, a GitHub outage) are logged and not shown.
- **The updater does not start.** `SPUUpdater.start()` runs on the main thread and can throw (for example on a malformed `SUPublicEDKey` or feed URL in a bad build). The app logs the error. The popover shows no update line. Settings shows "Updates are unavailable in this build" with the error, and the switch and **Check Now** are disabled. The rest of the app works as before.
- **First launch.** The first launch of the first release with Sparkle has no last-check time, so Sparkle checks at once. Later checks follow the daily interval.

### 3. When an update may install

- **The button.** **Update and Relaunch** is disabled, with the reason "after the scan finishes" or "after the clean finishes", while `CLIRunner` has a job running or queued in any lane.
- **The install.** `updater(_:shouldPostponeRelaunchForUpdate:untilInvoking:)` returns YES while `CLIRunner` has a job running or queued in any lane. The app's own runner state is the only gate. When the runner becomes idle, the app calls the stored handler once. While postponed, the line says "Waiting for the scan to finish".
- **No new jobs while an install waits.** `ScheduledWork.tick` (`app/Packages/DDCKit/Sources/DDCKit/ScheduledWork.swift:85`) gains a check at its top: while an install waits, it starts no scan and returns, and the scan waits for the next tick. Scan Now and the clean and free buttons are disabled with "Updating after this run".
- **No data-dir lock probe.** The app does not test `<data-dir>/lock`. Rust's `try_lock` never waits (`crates/ddc-core/src/store.rs:41-54`), so a probe holding the lock even for a moment could make a `ddc` started from Terminal exit 75 ("busy") for no reason. A `ddc` started from Terminal, or by the launchd agent of `ddc schedule install`, is not affected by the swap: Sparkle's installer (`SUPlainInstaller`) moves the old bundle aside, and a running process keeps its old inode until it exits. The next `ddc` starts the new binary.
- **Quit while an install waits.** Accepted. If the person quits the app while the install is postponed, Sparkle's installer finishes after the host has quit (`AppInstaller`, `finishInstallationAfterHostTermination`), as it does for any update that is ready at quit. A scan child that the quit leaves behind keeps running from the old inode, so nothing is corrupted. The update is installed, and the app is not relaunched.
- **Launch modes.** The updater is created at launch but started (`startUpdater`) only on the first `showStatusItem` effect of `LaunchMode`. A `--tick` launch never reaches `NSApplication`. A `--scheduled` launch that quits after its work never starts it; one that is promoted to normal starts it then.
- **Checks** are only network reads and may run during a scan.

### 4. Where the code goes

- **DDCKit:** `UpdateFlow`, a pure state machine (idle, available, information only, checking, downloading, extracting, waiting for idle, installing, failed, unavailable) with the events the adapter feeds it (including "runner idle" and "app quits") and the replies it returns. It reads "busy" from `CLIRunner`'s state only. It is unit tested.
- **App target:** `SparkleUpdater.swift`, the `SPUUserDriver` and `SPUUpdaterDelegate` adapter, with the Debug-only feed override of §1; the popover line; the Settings rows. `AppDelegate` starts the updater on `showStatusItem`.

### 5. The Settings switch

The switch "Check for updates daily" binds straight to `SPUUpdater.automaticallyChecksForUpdates`. It is not copied into `AppPreferences`: Sparkle stores it in the app's user defaults, and a second copy could drift. Off means no scheduled request. **Check Now** stays available and makes one request when pressed.

### 6. One artifact for both channels

The same notarized `DeepDriveCleaner-<v>.zip` is the cask's download and the Sparkle enclosure. `build-app.sh` zips it with `ditto -c -k --norsrc --noextattr --keepParent`: a signed bundle holds no resource fork, and the `com.apple.provenance` attribute macOS can put on every file a build writes cannot be removed with `xattr -cr`. With `--sequesterRsrc`, as Sparkle's documentation advises, that attribute landed in `__MACOSX/` and stopped `make dist` for 0.8.0. (Amended 2026-10-30.)

`build-app.sh --dist` refuses to build unless the app's `CURRENT_PROJECT_VERSION` is greater than the one in `project.pbxproj` at the last release tag. Sparkle compares `CFBundleVersion`, not the short version, so a build number that does not grow would never be offered.

`publish-release.sh` gains an appcast step, before the release exists:

1. Run `sign_update` on the zip. It reads the private key from the login keychain, so it runs unsandboxed, like `notarytool`.
2. Render `appcast.xml` from `packaging/sparkle/appcast.xml.tmpl` with one item: `sparkle:version` (the app's `CFBundleVersion`), `sparkle:shortVersionString`, `sparkle:minimumSystemVersion` (`MIN_MACOS` as three parts, `26.0.0`), `sparkle:fullReleaseNotesLink` (the release page), and an `enclosure` with the versioned download URL, `length`, `type="application/octet-stream"` and `sparkle:edSignature`. No `sparkle:releaseNotesLink`, so Sparkle makes no second request for notes. The root `<rss>` declares both the `sparkle` and the `ddc` namespaces. The rendered file must pass `xmllint --noout`.
3. Refuse to publish unless the new `CFBundleVersion` is greater than the one in the previous release's `appcast.xml` (when that release has one).
4. Refuse to publish an installable item when the app in the zip is signed by a team other than the previous release's. `--team-change` renders the item with `<sparkle:informationalUpdate/>` instead: a link, never an install. The appcast carries the team in a `ddc:team` element, which Sparkle ignores and the script reads back. (Sparkle itself would accept a team change when the EdDSA signature is valid, so this guard is ours.)
5. Pass `appcast.xml` to `gh release create` as a third asset, beside the zip and the tarball. The release becomes "latest" with its feed already in place, so the feed URL never returns 404 during a publish. The resume path and the "tap already current" exit upload a missing appcast.
6. After the tap is written, fetch the live `releases/latest/download/appcast.xml`, check it with `xmllint`, and compare its `sparkle:version` and `edSignature` with the rendered file. A mismatch fails the publish with a message. This network read runs only in the release script, never in tests.

For the minutes between the release and the tap write, the cask is one version behind the appcast. A `brew upgrade --greedy` in that window reinstalls the previous version over a Sparkle update; the next check offers the update again.

**Rollback.** The appcast holds one item, so Sparkle never offers an older version. The route back is `brew reinstall` of an older cask, or the zip from an older release page. A broken release is fixed by a newer one.

### 7. The EdDSA key

- The owner runs `generate_keys` once. The private key stays in the owner's login keychain. It is never in the repository, the vault, `.env` or CI. The owner exports a backup with `generate_keys -x` and keeps it offline.
- The public key goes into Info.plist as `SUPublicEDKey`.
- **Lost key.** The next release is signed with the same Developer ID, carries a new `SUPublicEDKey`, and its archive is signed with the new key. Sparkle accepts it because the Apple code signature still matches. That release must not also change the team.
- **Leaked key.** Rotate the same way at once. A leaked key alone cannot ship an update: the attacker also needs write access to the release, or a TLS path they control.
- **Team change.** Never in the same release as a key change. It ships as an information-only item (§6, step 4), and people install it by hand or with `brew upgrade --greedy`, granting Full Disk Access once more.

### 8. Signing Sparkle

In `build-app.sh`, before the CLI and the app are signed:

1. Delete `Sparkle.framework/Versions/B/XPCServices` and the `XPCServices` link (not needed outside the sandbox).
2. Sign, with the Developer ID, hardened runtime and timestamp, and Sparkle's own identifiers (no `--identifier`): `Versions/B/Autoupdate`, then `Versions/B/Updater.app`, then `Sparkle.framework`.
3. Then the CLI and the app, as today, and `codesign --verify --deep --strict`.
4. A new check: every Mach-O file in the bundle has the expected team and the hardened runtime flag. `make dist` fails otherwise. Its failure case is tested anywhere (`make app-test`) on a fixture signed ad hoc (`codesign -s -`); its pass case needs the Developer ID and runs on a real build.

Notarization covers the framework with the rest of the app.

### 9. Homebrew, next to Sparkle

- **The cask gains `auto_updates true`.** Current Homebrew still upgrades such a cask on a plain `brew upgrade` when the installed bundle's version is older than the tap's (`Cask#auto_updates_bundle_outdated?`, on by default; `HOMEBREW_NO_UPGRADE_AUTO_UPDATES_CASKS` turns it off), and skips it when the bundle is already current, as after a Sparkle update. So brew and Sparkle never both install the same version, and someone who never opens the app still updates with `brew upgrade`. With the opt-out set, or on an older Homebrew, `brew upgrade --greedy deepdrivecleaner` is the way. (Amended 2026-10-30 after reading Homebrew's `cask.rb`; the first text said `brew upgrade` always skips it.)
- **The `binary` links** point into `Contents/Helpers`, so `ddc` runs the new CLI after an in-place update.
- **Caskroom's version** stays at the one brew installed until brew next installs it. This is cosmetic: `brew list --versions` shows the old number. `brew uninstall --cask` removes the app by its path and the links by name, so it works. A `--greedy` upgrade to the version already running reinstalls it, harmlessly.
- **`uninstall quit:`** keeps its meaning: brew quits the app before an uninstall, a `--greedy` upgrade or a reinstall. A Sparkle update does not go through brew and does not use it.
- **`zap`** adds Sparkle's paths to `trash:`, keeping the list alphabetical: `~/Library/Caches/com.smartandpoint.deepdrivecleaner` (its download folder) and `~/Library/HTTPStorages/com.smartandpoint.deepdrivecleaner`. Its preference keys are in the app's preferences plist, which `zap` already trashes.
- **Ownership.** brew installs the app as the user, so Sparkle replaces it without a password. A copy an administrator dragged in for another user asks for an administrator through Sparkle's own authorization.
- **Grants and agents.** Sparkle puts the new bundle at the same path with the same bundle id and team, so the Full Disk Access grant matches. Sparkle relaunches the app, and with scheduled scans on, `heal()` re-registers the scheduler agent at launch (`app/DeepDriveCleaner/AppDelegate.swift:142-144`; §5 of the menu bar ADR). The login item is not re-registered: it survives because `SMAppService.mainApp` is keyed to the bundle at the same path. The manual check in the story confirms the grant, the agent and the login item on a real Mac.
- **The README** tells Homebrew users that the app updates itself, and that someone who uses only the CLI and never opens the app updates with `brew upgrade --greedy deepdrivecleaner`.

### 10. Network and privacy

This is the app's first network request of its own.

- **What.** One HTTPS GET for `appcast.xml` from github.com (redirected to GitHub's asset host): once at the first launch of the first release with Sparkle, then at most once a day, plus one on **Check Now**. On **Update and Relaunch**, the zip from the same hosts.
- **What GitHub sees.** The IP address, the time, the `User-Agent` and the system language. The app sets `userAgentString` to `DeepDriveCleaner`, with no version; Sparkle passes it to the archive download too (`SPUUpdateDriver` hands `_userAgent` to `SPUDownloadDriver`). `NSURLSession` adds `Accept-Language`. No headers of ours, no parameters, and system profiling is off.
- **What the maintainers see.** The release's download counter for `appcast.xml`: a total count, nothing per user.
- **The opt-out** is the switch of §5.
- **The README** gets a "Network" section that says the above; it is written during implementation. The website's privacy text is the website epic's to update.

### 11. The standalone CLI

There is no `ddc update`, no `latest.json` and no brew helper. A CLI self-update belongs to the epic [[epics/DDC-007/epic|Standalone CLI on macOS, Linux and Windows: portable core, release builds and installers]] and can read the same appcast later (Consequences).

## Rationale

1. **Sparkle solves the hard part.** The research note's main finding was that the relaunch, not the check, is hard. Sparkle's installer replaces the bundle after the app quits and relaunches it, handles quarantine and authorization, and checks the minimum macOS. We write an adapter and a release step instead of a helper.
2. **It covers every install.** A hand-dragged copy and a Homebrew copy update the same way. The brew helper covered Homebrew installs only.
3. **One artifact, two channels.** The zip brew downloads is the zip Sparkle downloads, with one notarization and one digest, and the feed ships with the release that it describes.
4. **The custom driver fits an `LSUIElement` app.** The owner wants the button in the popover and in Settings. Sparkle's standard window, even with gentle reminders, adds its own Install button and needs the activation policy changed so the window can come forward.
5. **The safety rules map onto Sparkle's hooks.** The postpone delegate holds the install while the app's runner has work; the launch mode decides when the updater starts. Both are small and testable in DDCKit.
6. **Homebrew's `auto_updates` is the documented contract** for an app that updates itself. It stops `brew upgrade` from fighting Sparkle.

## Alternatives Considered

### Alternative A: A detached `brew upgrade` helper (the first answer, route a1)

The app quits and leaves `brew update && brew upgrade --cask deepdrivecleaner` to a detached `deepdrivecleaner update --via-homebrew`, which reopens the app. Other installs see the version and the command.

**Pros**:
- No framework, no new key; brew stays the only installer.
- The cask keeps no `auto_updates`, so `brew upgrade` keeps working without `--greedy`.

**Cons**:
- Homebrew installs only; a hand-dragged copy gets a command to copy.
- Races: the cask's `uninstall quit:` quits the app while the helper runs, the scheduler may open the app mid-upgrade, and the helper must outlive both and reopen the right copy.
- A helper started by the app writes to `/Applications` and the Caskroom; whether macOS asks for App Management there is unproven.
- `brew update` can take minutes and fail on an unrelated tap, all behind one button.
- More code than the adapter: a lane kind, an `update.json` contract, a `latest.json` manifest, and a CLI subcommand with an HTTP client.

**Rejected because**: the owner chose Sparkle, and it is both narrower in coverage and larger in code than the Sparkle adapter.

### Alternative B: Notify only (route c)

The app shows "Version X is available" and the command or link; it installs nothing.

**Pros**:
- Smallest change; no installer at all.

**Cons**:
- The owner asked for **Update and Relaunch**.

**Rejected because**: it does not meet the owner's decision.

### Alternative C: Our own bundle swap (route a2)

**Pros**:
- No third-party framework.

**Cons**:
- Re-implements Sparkle: download verification, quarantine, an atomic swap, authorization for admin-owned copies, rollback.

**Rejected because**: Sparkle already does this and is maintained.

### Alternative D: Sparkle's standard user driver with gentle reminders

**Pros**:
- No user driver to write; Sparkle's tested UI.

**Cons**:
- Sparkle's window has its own Install, Skip and Remind buttons, so the popover button would be a second path into the same window.
- For a Dock-less app, Sparkle's documentation recommends switching the activation policy, badging the Dock and posting a notification: three behaviours this app does not have.

**Rejected because**: the custom driver gives one press in the popover and no window, for about fifteen small methods.

### Alternative E: Sparkle with automatic install (`SUAutomaticallyUpdate` on)

**Rejected because**: the owner decided that install happens only on the button.

## Consequences

**Positive**:
- Every copy, Homebrew or not, gets a one-press update with relaunch.
- Full Disk Access, the login item and the scheduler agent stay in place, because the path, bundle id and team do not change.
- The same zip and one release step serve brew and Sparkle.

**Negative / trade-offs**:
- A third-party framework in the app, with three signed binaries to keep in the signing order.
- A new secret, the EdDSA key, that only the owner holds. Losing both it and the Developer ID at once would strand existing installs on manual updates.
- The app makes a network request by default. The switch turns it off.
- `brew list --versions` shows an old version after a Sparkle update. `brew upgrade` updates the app only when its bundle is older than the tap (by default; `--greedy` where the opt-out is set).
- Users on 0.7.0 and earlier have no Sparkle: they reach the first Sparkle release through `brew upgrade` or by hand.
- The appcast holds only the newest item. A Mac below a release's minimum macOS sees no update rather than an older compatible one; that is today's behaviour too. Rollback is by hand: `brew reinstall` of an older cask, or an older release page.
- A quit while an install waits installs the update without a relaunch (§3).

**What changes in code / process**:
- `app/DeepDriveCleaner.xcodeproj`: the Sparkle package; Info.plist keys.
- `app/DeepDriveCleaner/`: the Sparkle adapter, the popover line, the Settings rows, the start on `showStatusItem`.
- `app/Packages/DDCKit`: `UpdateFlow` and its tests; the install-pending check in `ScheduledWork.tick`.
- `scripts/build-app.sh`: XPC removal, Sparkle signing, the per-binary check, the zip without extended attributes, the build-number check against the last tag.
- `scripts/publish-release.sh` and `packaging/sparkle/appcast.xml.tmpl`: `sign_update`, the appcast as a release asset, the version and team guards, the check of the live feed.
- `packaging/homebrew/deepdrivecleaner.rb.tmpl`: `auto_updates true`, the `zap` paths.
- `README.md`: "Network", the update text under Install and Full Disk Access.
- `ops/`: a note for the first real pair of Sparkle releases (the first one ships the updater; the second is the first update through it).
- The owner: `generate_keys` once, and an offline backup.
- ADR [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]: its Context item 4 ("no Sparkle, updates through `brew upgrade`") no longer holds for this app; its §8 records the amendment.
- Later, the epic [[epics/DDC-007/epic|Standalone CLI on macOS, Linux and Windows: portable core, release builds and installers]]: a CLI self-update can read the same appcast, but the appcast's enclosure is the app zip, so the CLI would need its own items or a separate feed for its tarballs.

## Open Questions

- **Authorization prompts.** Whether macOS asks for App Management, or Sparkle for a password, when the copy belongs to another administrator. The manual test covers a user-owned copy; an admin-owned one is noted if seen.
- **homebrew/cask after 2026-11-16.** The coordinator's note that homebrew/cask will expect `auto_updates` for self-updating apps is not on the Acceptable Casks page as read on 2026-10-29. The cask is in our own tap, so this matters only if it moves to homebrew/cask.
- **The `zap` paths.** The two Sparkle paths are from Sparkle's defaults; the plan confirms them on a built app.

## References

- Research [[research/update-check-and-update-in-place|Update check and update in place]]
- ADR [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]], Context item 4, §5 and §8
- Story "Sparkle updates: check daily, Update and Relaunch, and the cask's auto_updates" (epic DDC-002, "Menu bar app")
- `app/DeepDriveCleaner/Info.plist`, `app/DeepDriveCleaner/main.swift`, `app/Packages/DDCKit/Sources/DDCKit/LaunchMode.swift`, `app/Packages/DDCKit/Sources/DDCKit/CLIRunner.swift`, `app/Packages/DDCKit/Sources/DDCKit/ScheduledWork.swift`, `app/DeepDriveCleaner/AppDelegate.swift`, `crates/ddc-core/src/store.rs`
- `scripts/build-app.sh`, `scripts/publish-release.sh`, `packaging/homebrew/deepdrivecleaner.rb.tmpl`
- Sparkle documentation: Customizing Sparkle (Info.plist keys), Publishing an update (`sign_update`, appcast elements), Sandboxing (removing the XPC services, manual signing), EdDSA keys and key rotation, Gentle reminders, `SPUUpdater`, `SPUUpdaterDelegate`, `SPUUserDriver`
- Homebrew Cask Cookbook: `auto_updates`, `uninstall quit:`; `brew upgrade --greedy`

---

*Last updated: 2026-10-29*

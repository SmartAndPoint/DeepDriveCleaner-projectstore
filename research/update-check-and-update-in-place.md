---
type: research
slug: "update-check-and-update-in-place"
title: "Update check and update in place"
status: final
date: 2026-10-29
authors: ["Evgenii Konev"]
tags: [idea, updates, distribution, homebrew, network, privacy, settings]
review_status: reviewed
reviewed_at: 2026-10-29
---

# Update check and update in place

## Question

An idea from the owner, 2026-10-29, recorded for later:

- While the app runs, it checks whether a newer release exists. If one does, it notifies the user.
- The popover and Settings get an **Update and Relaunch** button.
- Settings get an **Update automatically** checkbox, which the user can switch off.
- The check reads the GitHub release, which has the URLs of the assets. If the version is higher, the app updates itself, or updates through Homebrew on the Mac. On Linux and Windows, where only the CLI exists (epic "Standalone CLI on macOS, Linux and Windows: portable core, release builds and installers"), it probably updates itself.

The owner: "we need to think it through, but later". This note records the idea and the questions that decide it. It is not a decision.

## TL;DR

- **The first network request the product makes of its own.** Today the app and the CLI make no network requests: the code has no URLSession and no HTTP client. (The Free action's `brew cleanup` is Homebrew's process, with Homebrew's analytics unless the user turned them off; `BREW_ENV` in `free/brew.rs` does not set `HOMEBREW_NO_ANALYTICS`. That is a separate small fix.) An update check changes that. The README must say so and name who receives the request, Settings must switch it off, and the request must carry nothing but what it needs. A daily check from every install still lets the receiver count installs by IP, whatever the request says.
- **Proposed defaults:** *check* on, announced by one line at first launch; *install* off. That follows the product's stance that the app informs and people act (no unattended clean). The owner asked for an "Update automatically" checkbox the user can switch off, which reads as install on by default. This is the first question for the owner.
- **The hard part is not the check, it is the relaunch.** Spawning `brew upgrade --cask` from the app kills the app half way: the cask's `uninstall quit:` quits it, the brew child loses its parent, and nothing opens the new version until the next scheduler tick or login (ADR "Menu bar app: Swift shell over the ddc CLI", §5 and §8). Every in-app route needs a detached helper that outlives the app and relaunches it, or Sparkle, which has one.
- **Routes on the Mac, compared below:** (a1) for a Homebrew install, a detached `brew update && brew upgrade --cask` that reopens the app, with other installs falling back to (c); (a2) our own verified bundle swap; (b) Sparkle with `auto_updates true` in the cask; (c) notify only. Sparkle solves the relaunch, Gatekeeper quarantine, minimum macOS, release notes and signing; its cost is a framework, an appcast, and Homebrew users who must pass `--greedy` to `brew upgrade` for the app.
- **The check needs a manifest, not only the latest tag.** The cask requires macOS 26 and Apple Silicon. A release that raises either must not be offered to a Mac that cannot run it. `releases/latest` has no such field. A small `latest.json` attached to each release (or Sparkle's appcast) carries the version, the minimum macOS, the architectures and the notes URL.
- **Verification is two things.** A SHA-256 from the same origin as the download proves the file arrived whole, not who made it. Authenticity on the Mac is the Developer ID signature checked against the running app's designated requirement; on Linux and Windows, the build provenance attestations the standalone CLI epic plans to publish (it is planned; today's release has neither `SHA256SUMS` nor attestations).

## Method

Reading only:
- the code (no HTTP client in `Cargo.toml` or the Swift sources);
- the cask template (`packaging/homebrew/deepdrivecleaner.rb.tmpl`: `uninstall quit:`, `depends_on macos: :tahoe`, `depends_on arch: :arm64`, `binary` links into the bundle, no `auto_updates`);
- `scripts/publish-release.sh` (the assets: the app zip and the CLI tarball);
- ADR "Menu bar app: Swift shell over the ddc CLI", §5 (the scheduler agent) and §8 (signing, the cask, the team);
- epic "Standalone CLI on macOS, Linux and Windows: portable core, release builds and installers" (`SHA256SUMS`, attestations, `install.sh` that upgrades when run again);
- Homebrew's documentation on `auto_updates` and `--greedy`, and Sparkle's on EdDSA appcasts.

Nothing was prototyped.

## Findings

### The current release path

- `publish-release.sh` creates a GitHub release with `DeepDriveCleaner-<v>.zip` (the notarized, stapled app) and `deepdrivecleaner-<v>-aarch64-apple-darwin.tar.gz` (the CLI), then writes the version and the sha256 into the Homebrew tap.
- An upgrade is `brew upgrade --cask deepdrivecleaner`. Its `uninstall quit:` quits the app. §5 says the scheduler agent opens the new app at its next hourly tick, and also that the app re-registers "an agent dropped when `brew` replaced the bundle" at its next launch. If the agent is dropped, nothing reopens the app before the user does. Which one holds was not checked here; an update route should not rely on the tick.
- The cask links `deepdrivecleaner` and `ddc` from `/opt/homebrew/bin` into the bundle. The CLI's own agent from `ddc schedule install` resolves its path (`schedule.rs` canonicalizes `current_exe`), so the launchd `ProgramArguments` hold the bundle path itself, `/Applications/DeepDriveCleaner.app/Contents/Helpers/deepdrivecleaner`; `homebrew_stable_path` (`launchd.rs`) maps only binaries under `Caskroom` or `Cellar`. So every route, brew included, must land the new bundle at that path with `Contents/Helpers` inside it. A swap made of two renames (old aside, new in) leaves a gap in which `ddc schedule status` reports the program missing and a tick fails; item 5 avoids it.

### The check

- **Where.** Two cheap sources:
  - `HEAD https://github.com/SmartAndPoint/DeepDriveCleaner/releases/latest`: the redirect's `Location` names the tag, and the web route has no API rate limit;
  - `GET https://api.github.com/repos/SmartAndPoint/DeepDriveCleaner/releases/latest`: `tag_name` and the assets. It needs a `User-Agent` header, and allows 60 unauthenticated requests an hour per IP. That limit is shared by everyone behind the same NAT, so a failure is normal and Settings shows "Last checked: …".

  Neither says which macOS a release needs, so a manifest is the better source (TL;DR). A `latest.json` attached to each release is reached at `https://github.com/SmartAndPoint/DeepDriveCleaner/releases/latest/download/latest.json`, GitHub's stable URL for the latest release's asset: no API, no rate limit, published by the same `gh release create` the release script already runs, and it cannot drift from the release it describes. The tap's cask file (`version`, `depends_on macos:`) is a second existing source. The `v` is stripped from the tag before the version comparison.
- **What it reveals.** A `User-Agent` such as `DeepDriveCleaner/<version>` tells GitHub which version asks. The note proposes a constant `DeepDriveCleaner` with no version, and no other header or parameter. Sparkle's default User-Agent carries the app's version too; it can be set.
- **When.** At most once a day, never during a scan or a clean, and not in the `--scheduled` mode that quits after its scan (§5).
- **Who does it.** Either the app's Swift code, or a `ddc update` subcommand (`ddc update --check --json` for the check) that the app runs like every other CLI call. The subcommand keeps one implementation for the app and the standalone CLI, and adds an HTTP client to the core. That choice belongs in the ADR.

### Installing on the Mac

1. **How the app knows Homebrew installed it.** Any version folder under `Caskroom/deepdrivecleaner/` in the Homebrew prefix (`/opt/homebrew`), found with the same location probe as `brew cleanup` (`free/probe.rs`). A copy dragged in by hand has none. After a Sparkle update (route b), the folder's version is older than the running app.
2. **The relaunch.** The process that replaces the app cannot be the app. It is started detached (its own session, output to a log) or as a one-shot launchd job, outlives the app, and opens the new app at the end.
   - **(a1) Homebrew.** The process is a short shell line: `brew update`, then `brew upgrade --cask deepdrivecleaner`, then open the app by bundle id. brew quits the app, moves the bundle and rolls back its own failures. `brew update` is needed explicitly: `HOMEBREW_NO_AUTO_UPDATE` would hide the new version, and the default auto-update runs at most once every 24 hours (`HOMEBREW_AUTO_UPDATE_SECS`), so a user who ran brew today would get "nothing to upgrade" and a relaunch of the old app. The result is judged by the installed version afterwards, not by brew's exit code. The tap update is why this can take a minute, and the popover says so.
   - **(a2) A bundle swap.** Our own helper, signed with the app and shipped in it, downloads, verifies (item 4), and swaps. Items 3 to 7 are its costs. It needs write access to `/Applications` and to the old bundle: a standard user, or a bundle another admin owns, cannot; Sparkle has an authorised-install path for that, a helper would need one. A failure is logged and shown by the app at its next launch.
   - A cask `postflight` that reopens the app would be cheaper for (a1), but Homebrew's style discourages it and it would also run on a plain `brew upgrade` the user did not start from the app. Not proposed.
3. **Quarantine and App Translocation (a2).** URLSession writes the quarantine attribute only when the app opts in (`LSFileQuarantineEnabled` in Info.plist, or a per-file quarantine property); this app does not, and a `ddc update` the app spawns inherits that. Setting the attribute is a choice. With it, a bundle opened from a temporary location is translocated by Gatekeeper, and `Bundle.main` paths, `HelperLocator`, `--tick` and the `SMAppService` registration break, so the helper must move the new bundle into place before the first launch, verify it, and clear the attribute only after the verification. Without it there is no translocation, but also no Gatekeeper assessment at first launch: item 4 is then the only gate. Sparkle handles both.
4. **Verification of a downloaded app.** `codesign --verify --deep --strict` and a match of the new bundle's designated requirement against the running app's: the same identifier and the same Team ID. A stapled ticket lets `spctl --assess` pass offline.
5. **Races (a2).** The scheduler agent may start the app with `--scheduled` while the helper replaces it. The helper takes the data-dir lock (`<data-dir>/lock`, `store.rs`: scans share it, a clean takes it exclusively) exclusively. The lock does not wait (`try_lock`), so the helper polls until no scan or clean runs, and a scan started during the swap gets "another ddc scan or clean is running" and is skipped until the next tick. The lock stops a scan, not a launch: a `--tick` or `--scheduled` launch during the swap is safe only if the swap is atomic: `rename(2)` cannot replace a non-empty folder, so the swap is `renameatx_np` with `RENAME_SWAP` (atomic on APFS), and the swapped-out bundle is the old copy item 7 keeps. After the swap, the `SMAppService` agent is re-registered (its plist lives in the bundle).
6. **No team migration through an update.** The designated requirement includes the Team ID (§8, "Team"). An update signed by another team fails the check above, and Full Disk Access is lost anyway. The planned switch to an organization team is a manual step with its own release notes, never an automatic update.
7. **Rollback (a2).** The old bundle is kept beside the new one until the new app has started once and answered a new `--version` flag (today the only flag that exits before AppKit is `--tick`, which opens the app, `main.swift`; `--scheduled` is read later by `LaunchMode`). If it does not start, the helper puts the old one back. While two bundles with one bundle id are on disk, the fallback agent's `open -b` may resolve to the old copy; the ADR has to say where the old copy waits. In (a1), brew rolls back its own failed upgrade. Downgrades are never offered; a broken release is fixed by a newer one.
8. **Release notes.** The popover's update line links to the release page, or shows the manifest's short notes. Settings shows the same.
9. **The cask and `auto_updates`.** With route (b), the cask says `auto_updates true`. `brew upgrade` then skips the app unless `--greedy`, and Homebrew's record stays at the old version until then. The app updates itself, so that only affects `brew list --versions`. A Homebrew user who uses only the CLI would never get it through `brew upgrade` without `--greedy` either: route (b) means the README tells them so.

### The standalone CLI

The epic "Standalone CLI on macOS, Linux and Windows: portable core, release builds and installers" (planned) plans the parts:

- `install.sh` (and the Windows script) that upgrades in place when run again, verifying the archive against `SHA256SUMS`;
- build provenance attestations on the Linux and Windows archives; the macOS CLI is Developer ID signed today.

So `ddc update` can be "run the installer's logic for this platform": download, check `SHA256SUMS`, check the attestation or the signature, replace the binary. On macOS and Linux that is a `rename` over the old file. On Windows a running `.exe` cannot be overwritten, so the old one is renamed aside first and the new one put in its place. A CLI that a package manager installed (Homebrew, winget, cargo) only says which command updates it.

### Questions for the owner, before an ADR

1. **Defaults.** Check on with the first-launch line, install off (proposed); or install on, as the owner's wording implies. With (b), Sparkle's own default is a third answer: no checks until it asks the user at the second launch, and no automatic install.
2. **Route on the Mac.** (a1) brew for a Homebrew install and (c) for the rest, (a2) our own bundle swap, (b) Sparkle, or (c) notify only, with the brew command to copy.
3. **The manifest.** A `latest.json` attached to each release by `publish-release.sh` (proposed), an appcast (with (b)), or the GitHub tag alone, accepting that a release that drops a macOS version must not be published under the same cask. A copy on deepdrivecleaner.app can come later, once the site is public.
4. **Where the check lives.** In the app, or a `ddc update` subcommand shared with the standalone CLI.

## Comparison

| Option | Pros | Cons |
|--------|------|------|
| (a1) A detached `brew update && brew upgrade --cask`, then reopen; other installs get (c) | Homebrew's record stays right; brew does the quit, the move and the rollback; a few lines | Only for Homebrew installs; the tap update is slow; brew's analytics see the command |
| (a2) Our own verified bundle swap | No third-party framework; works for any install | The helper, its signing, quarantine handling, permissions on `/Applications`, rollback and the lock are ours to write and test |
| (b) Sparkle everywhere, `auto_updates true` in the cask | Solves relaunch, quarantine, minimum macOS, release notes, EdDSA signing and critical updates; the common pattern outside the App Store | A framework and an appcast to publish; `brew upgrade` skips the app without `--greedy`; one more key to keep (EdDSA) |
| (c) Notify only: the popover shows the new version, its notes and the command (`brew upgrade --cask deepdrivecleaner`) | Smallest; nothing replaces the app; no relaunch problem | Not the "Update and Relaunch" button the owner asked for |

Either way the check itself is the same small piece, and (c) is a step towards (a1), (a2) or (b), not a dead end.

## Conclusion

The idea is recorded, and the owner will decide later. When it is picked up, the four questions above become an ADR: the Homebrew-or-Sparkle choice first, with an amendment to ADR "Menu bar app: Swift shell over the ddc CLI" (§8 for the cask and the network request, §5 for who relaunches the app after an update). Then stories follow in the epic "Menu bar app" for the app, and in the standalone CLI epic for `ddc update`.

## The owner's answers (2026-10-29)

Asked in the session that picked the idea up:

1. **Defaults.** The check is on: at most once a day, shown in the popover and Settings, and Settings can switch it off. Install happens only when the person presses **Update and Relaunch**, never on its own.
2. **Route on the Mac.** First (a1), a detached `brew upgrade` for Homebrew installs; changed the same day to **(b), Sparkle 2 for every install**, with `auto_updates true` in the cask.
3. **The manifest.** Sparkle's `appcast.xml`, attached to each release and read at `releases/latest/download/appcast.xml`. No `latest.json`.
4. **Where the check lives.** In the app, through Sparkle. No `ddc update` for now; a CLI self-update is the standalone CLI epic's.

The decision is the ADR [[adr/updates-through-sparkle-alongside-the-homebrew-cask|Updates through Sparkle, alongside the Homebrew cask]].

## References

- `scripts/publish-release.sh`, `packaging/homebrew/deepdrivecleaner.rb.tmpl`
- ADR [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]], §5 and §8
- Epic [[epics/DDC-007/epic|Standalone CLI on macOS, Linux and Windows: portable core, release builds and installers]]
- Homebrew Cask Cookbook: `auto_updates`; `brew upgrade --greedy`
- Sparkle documentation: EdDSA signing, appcast, installer
- GitHub REST API: "Get the latest release"
- [[research/distribution-through-agents|Distribution through agents]]

---

*Last updated: 2026-10-29*

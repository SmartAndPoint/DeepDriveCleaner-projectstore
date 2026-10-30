---
type: runbook
slug: "first-sparkle-releases-shipping-the-updater-then-proving-it"
title: "First Sparkle releases: shipping the updater, then proving it"
status: draft
date: 2026-10-30
authors: ["Evgenii Konev"]
tags: [release, sparkle, homebrew, updates]
---

# First Sparkle releases: shipping the updater, then proving it

## Purpose

The release that first carries Sparkle (0.8.0) cannot update itself into existence: copies on 0.7.0 have no updater. Its own update path is first exercised when the next release ships. This runbook covers both releases, and what changed in every release from now on. It follows the ADR [Updates through Sparkle, alongside the Homebrew cask](../adr/updates-through-sparkle-alongside-the-homebrew-cask.md) and the story [Sparkle updates: check daily, Update and Relaunch, and the cask's auto_updates](../epics/DDC-002/stories/story-sparkle-updates-check-daily-update-and-relaunch-and-the-casks-autoupdates.md) (DDC-002).

## Prerequisites

- [ ] The Sparkle EdDSA private key is in the owner's login keychain, and its offline backup exists (`generate_keys -x`, done 2026-10-30).
- [ ] `CURRENT_PROJECT_VERSION` in `app/DeepDriveCleaner.xcodeproj/project.pbxproj` is greater than at the last tag. `scripts/check-build-number.sh` refuses the build otherwise. Raise it in every release commit from now on, together with `MARKETING_VERSION` and the Cargo version.
- [ ] CI Gate is green for the release commit.

## Steps

### 1. The first Sparkle release (0.8.0)

Run the release as always, unsandboxed, from a fresh clone (the keychain is read by notarytool, `generate_keys -p` and `sign_update`):

```bash
git clone https://github.com/SmartAndPoint/DeepDriveCleaner.git <scratch>/pub-080
cd <scratch>/pub-080 && set -a && . <code repo>/.env && set +a
make dist && ./scripts/publish-release.sh
```

`publish-release.sh` signs the zip with `sign_update`, renders `appcast.xml`, attaches it to the release with the zip and the tarball, writes the tap's cask, then fetches the live feed and compares its version and signature. There is no previous appcast for this first release, so the build-number guard against the feed has nothing to compare; `check-build-number.sh` still compares against the last tag.

Copies on 0.7.0 reach 0.8.0 the old way: `brew upgrade` (current Homebrew upgrades an `auto_updates` cask whose bundle is older than the tap), or a download from the release page.

### 2. Check the first release

- [ ] `https://github.com/SmartAndPoint/DeepDriveCleaner/releases/latest/download/appcast.xml` serves one item with the new version and an `edSignature`.
- [ ] On the reference machine after `brew upgrade`: Settings → General shows Check for updates daily, on, and Check Now says the app is up to date.

### 3. The second Sparkle release (0.8.1 or 0.9.0)

Release as in step 1. Then, on the reference machine with 0.8.0 installed, run the story's manual checks, now listed in [Owner checks: manual acceptance across stories](./owner-checks-manual-acceptance-across-stories.md) once the story closes:

- [ ] A Homebrew copy: the popover shows the new version; Update and Relaunch installs it and the app comes back; Full Disk Access, the scheduled scan, Login Items and `ddc --version` are as before.
- [ ] A copy dragged into /Applications by hand: the same.
- [ ] After the Sparkle update, `brew upgrade` leaves the app alone.

Before the second release, the same path can be rehearsed on a Debug build against a local feed (`scripts/debug-feed.sh`, the `DebugFeedPort` default), as the story's Debug AC describes.

## Verification

- [ ] The live-feed check at the end of `publish-release.sh` passed.
- [ ] `brew info --cask deepdrivecleaner` shows the new version and `auto_updates`.

## Rollback

- A broken feed: fix and rerun `./scripts/publish-release.sh`. It resumes at "tap current", uploads a missing `appcast.xml`, and checks the live feed again.
- A release that breaks the app: the appcast holds one item, so Sparkle cannot go back. Publish a fixed release with a higher build number. People who cannot wait run `brew reinstall --cask deepdrivecleaner` after the fix, or download an older zip from the release page.
- Never delete a release whose appcast copies have already read, without publishing a newer one first: `releases/latest` would point at the older release, whose build number is lower, and Sparkle offers nothing.

## Common Issues

### `check-build-number.sh` refuses the build

`CURRENT_PROJECT_VERSION` was not raised. Raise it in the release commit.

### `generate_keys -p` does not match the zip's `SUPublicEDKey`

The keychain holds another key, or Info.plist was changed. Do not publish. Restore the key from the offline backup with `generate_keys -f <file>`.

## References

- ADR "Updates through Sparkle, alongside the Homebrew cask"
- Epic DDC-002 "Menu bar app"
- Homebrew `Library/Homebrew/cask/cask.rb`: `outdated_version`, `auto_updates_bundle_outdated?`

---

*Last updated: 2026-10-30*

---
type: story
id: "story-homebrew-tap-and-the-v010-release"
epic: "DDC-001"
title: "Homebrew tap and the v0.1.0 release"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-17
updated: 2026-10-18
external_refs: {}
tags: [release, homebrew, distribution]
code_refs: []
specs: []
started_at: "2026-10-17T15:49:33.544Z"
closed_at: null
plan_updated_at: "2026-10-18T05:26:25.445Z"
---

# Homebrew tap and the v0.1.0 release

| Field | Value |
|---|---|
| **Epic** | [DDC-001](../epic.md) |
| **Status** | in-progress |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

Today `ddc` is installed with `make install` into `~/.cargo/bin`, which needs a Rust toolchain and a clone. Anyone who reads the public repository should be able to install it with one command: `brew install --cask smartandpoint/tap/deepdrivecleaner`. This story publishes the first release, `v0.1.0`, adds a cask to the existing SmartAndPoint tap (which already ships the `grammarhelper` cask) and moves the reference machine from the cargo install to the Homebrew one.

The tap follows the pattern it already uses for GrammarHelper: the source repository owns the template and the publish script; the tap only receives the rendered file with the version and the checksum substituted ("do not edit in the tap by hand"), written through the GitHub API. Signing follows GrammarHelper too: the release binary is signed with the SmartAndPoint Developer ID and notarized by Apple, so it runs on any Mac and keeps its Full Disk Access grant across `brew reinstall` and rebuilt binaries at the same path. The grant does not survive `brew upgrade`: TCC keys a command-line tool by its absolute path as well as its signature, and Homebrew installs every version in a new `Caskroom/deepdrivecleaner/<version>/` directory.

A cask, not a formula: Homebrew's rules put prebuilt binary-only software in casks, and a formula without a bottle counts as a source build, so `brew install` runs the Command Line Tools checks and refuses on a machine with outdated tools (the reference machine has CLT 16.3 on macOS 26: `formula_installer.rb` calls `perform_build_from_source_checks` unless it pours a bottle).

## Decomposition

- [ ] `packaging/homebrew/deepdrivecleaner.rb.tmpl`: cask template with `version`/`sha256` placeholders, the release tarball URL, `name`, `desc` (≤ 80 chars, no leading article or name), `homepage` `https://github.com/SmartAndPoint/DeepDriveCleaner`, `livecheck` on GitHub releases, `depends_on arch: :arm64`, `binary "deepdrivecleaner"` and `binary "deepdrivecleaner", target: "ddc"`, no `uninstall` stanza (Homebrew runs it on upgrades too, and the background scan should survive them), `zap` with `launchctl:` for the agent and `trash:` for the plist, data and logs, and caveats (Full Disk Access for `#{staged_path}/deepdrivecleaner`, re-add it after `brew upgrade`, `ddc schedule install`, `ddc schedule uninstall` or `--zap` before uninstalling)
- [ ] `make dist` (`scripts/build-dist.sh`): reads the version from `cargo pkgid`; deletes `dist/` first, so a failed run leaves nothing `publish-release.sh` can pick up; refuses a dirty tree (including untracked files); builds with `cargo build --release --locked --target aarch64-apple-darwin -p ddc-cli` under the `rust-toolchain.toml` pin, with `--remap-path-prefix=$HOME=~` and a check that the binary holds no `$HOME`; checks `lipo -archs` says `arm64` and `--version` prints the version; signs with `codesign --force --timestamp --options runtime --identifier com.smartandpoint.deepdrivecleaner --sign "$DDC_SIGN_IDENTITY"`; verifies with `codesign --verify --strict` and the team id; notarizes a zip of the binary with `xcrun notarytool submit --keychain-profile "$NOTARY_PROFILE" --wait` and fails with Apple's log unless the status is `Accepted`; checks `codesign --verify -R=notarized --check-notarization`, retried every 20 s for up to five minutes (`NOTARY_CHECK_ATTEMPTS`) while the ticket reaches Apple's lookup service (the check `brew audit` applies to cask binaries; `spctl --type execute` rejects every command-line tool as "not an app"); packs `dist/deepdrivecleaner-<version>-aarch64-apple-darwin.tar.gz` (binary, `LICENSE.md`, `README.md`) with `COPYFILE_DISABLE=1 tar --no-xattrs --no-mac-metadata --uid 0 --gid 0 --uname root --gname wheel` and checks the listing is exactly those three files; records the built commit in `dist/BUILD_COMMIT`. `DDC_SIGN_IDENTITY` (the certificate's SHA-1 from `security find-identity -v -p codesigning`, not its name: the keychain holds two Developer ID identities) and `NOTARY_PROFILE` are read from a local, gitignored `.env`; a committed `.env.example` lists the two keys with no values; without them `make dist` refuses
- [ ] `scripts/publish-release.sh [--dry-run]`: reads the version from `cargo pkgid` and never takes it as an argument; refuses a missing `docs/releases/<version>.md`, a dirty tree and a HEAD that is not on `main` or not pushed (`git fetch origin` and `HEAD == origin/main`); reads the tag `v<version>` from origin with `git ls-remote`, never from a local tag. **New release** (no release `v<version>` yet): takes `dist/<tarball>`; refuses a `dist/BUILD_COMMIT` that is not HEAD and a tag `v<version>` that already exists on origin; creates the release with `gh release create v<version> dist/<tarball> --target <built commit>` and notes rendered from `docs/releases/<version>.md` plus a generated block (commit, build command, toolchain, signing, SHA-256); then checks the uploaded asset's digest and that the tag on origin points at the built commit. **Resume** (the release exists: the asset went up, the cask did not): downloads the asset from the release instead of taking `dist/`, which a later `make dist` would have rebuilt with another digest; refuses a release it cannot read (only gh's "release not found" means there is none), a draft without the asset (gh deletes its own draft when an upload fails, so this one was left by hand; the message suggests `gh release delete --cleanup-tag`), a published release without the asset (the message says not to delete it: a cask may point at it), an asset GitHub reports no digest for, and a download that does not match the release's digest; takes the built commit from the tag on origin and refuses unless HEAD descends from it and `git diff --no-renames --name-only` since it lists only `packaging/homebrew/` or the publish script (a cask fix is a commit); writes nothing to the release repository. Both paths check the tarball before anything from it runs (its digest, then the signature and the SmartAndPoint team, then the version the binary prints), render the cask, read the tap's current file and, as the last step, write `Casks/deepdrivecleaner.rb` to `SmartAndPoint/homebrew-tap` through `gh api PUT repos/…/contents/…` with the commit message `deepdrivecleaner <version>`, the author and committer set explicitly and the current file's blob `sha` when the cask already exists; a tap whose file already has the rendered cask's blob sha is left alone. Never a `git push` from the local tap clone, which only `brew update` touches. `--dry-run` runs every check, renders the cask and the notes, prints them with what the release and tap writes would do (create, update or nothing) and publishes nothing; the dry run is what gets reviewed before the real run
- [ ] `ddc schedule install`: when the running binary lives in a Homebrew version directory (cask `<prefix>/Caskroom/<token>/<version>/<file>` → `<prefix>/bin/<file>`; formula `<prefix>/Cellar/<name>/<version>/bin/<file>` → `<prefix>/opt/<name>/bin/<file>`) and the stable link resolves to it, write the link into the plist, so the agent survives `brew upgrade`; `ddc schedule status` compares canonical paths, so the link is recognized as this binary
- [ ] README: install with Homebrew first; `make install` stays as the development path; the Full Disk Access paragraph names the Caskroom binary and says the grant must be re-added after `brew upgrade` (new path) but is kept across `brew reinstall`
- [ ] Publish `v0.1.0`: `make dist`, `scripts/publish-release.sh --dry-run`, then the real run after approval
- [ ] Reference machine: `ddc schedule uninstall`, `~/.cargo/bin/cargo uninstall ddc-cli`, remove `~/.cargo/bin/ddc` and both stale Full Disk Access entries (`~/.cargo/bin/ddc` and `~/.cargo/bin/deepdrivecleaner`), `brew install --cask smartandpoint/tap/deepdrivecleaner`, `ddc schedule install --threshold 10G`, grant Full Disk Access to the Caskroom binary

## Implementation Plan

Route from `projectstore:planner` (the agent's path rule belongs next to `inside_cargo_target` in `launchd.rs`, the seam the story "Scheduled background scan" created) and three `projectstore:critic` rounds (tap written through `gh api`, tag at the built commit, resume on an identical asset, Developer ID signing, path-keyed Full Disk Access, formula replaced by a cask).

1. **`crates/ddc-cli/src/launchd.rs`**: `homebrew_stable_path` (Caskroom → `<prefix>/bin`, Cellar → `<prefix>/opt/<name>/bin`, path arithmetic only), `same_binary`, `scheduled_program` (the stable link only when it resolves to the running binary), `program_state`; `install` writes `scheduled_program(exe)`. Tests in the module's `tests` with real temp trees and relative links as Homebrew makes them.
2. **`crates/ddc-cli/src/cmd/schedule.rs`**: `status` reports `program_state`.
3. **`packaging/homebrew/deepdrivecleaner.rb.tmpl`**: the cask.
4. **`scripts/build-dist.sh`** (`make dist`), `.env.example`, `.gitignore` (`/dist/`, `.env`): build, sign, notarize, `codesign --check-notarization` (retried), pack, `BUILD_COMMIT`.
5. **`scripts/publish-release.sh`**: guards; the new-release path (tarball from `dist/`, release at the built commit, tag checked on origin) and the resume path (asset from the release, built commit from the tag on origin, bounded diff); tarball checks; notes from `docs/releases/<version>.md`; cask through `gh api`.
6. **`README.md`**, **`docs/releases/0.1.0.md`**.
7. Commit and push; `make dist`; `--dry-run` and refusal transcripts in a scratch clone; the real run after approval; then the reference machine.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test: a cask binary at `<prefix>/Caskroom/deepdrivecleaner/<v>/deepdrivecleaner` linked from `<prefix>/bin` and a formula binary at `<prefix>/Cellar/deepdrivecleaner/<v>/bin/deepdrivecleaner` with `<prefix>/opt/deepdrivecleaner` pointing at that version are written into the plist as their stable links; any other location (no link, a link to another version, a dangling link, not a Homebrew directory) as the canonical executable; `status` treats the stable link as this binary
- [ ] `make dist` produces a tarball whose listing is exactly `deepdrivecleaner`, `LICENSE.md`, `README.md`; `codesign -dvv` on the binary shows `Authority=Developer ID Application`, the team id and the `runtime` flag; the notarization status is `Accepted` and `codesign --verify -R=notarized --check-notarization` succeeds
- [ ] `scripts/publish-release.sh --dry-run` prints the rendered cask and notes and creates no release or tap commit; transcripts show it refusing a dirty tree, a HEAD that differs from `origin/main`, a missing `docs/releases/<version>.md`, a `dist/BUILD_COMMIT` that is not HEAD when no release exists, a tag `v<version>` on origin without a release, and a resume whose diff since the tag lists a file outside `packaging/homebrew/` (the last two in a scratch clone whose `origin` is a local bare repository; the resume refusal needs the published release and is recorded after it)
- [ ] GitHub release `v0.1.0` of SmartAndPoint/DeepDriveCleaner carries the tarball, its tag points at the built commit, and the SHA-256 in the tap's `Casks/deepdrivecleaner.rb` matches it (`brew fetch --cask` verifies)
- [ ] On the reference machine, with the cargo install removed: `brew install --cask smartandpoint/tap/deepdrivecleaner` installs `deepdrivecleaner` and `ddc` into `/opt/homebrew/bin`, `ddc --version` prints `0.1.0`, `codesign -dvv` on the installed binary still shows `Authority=Developer ID Application` and the team id (Homebrew did not re-sign it), and `brew style` reports no offenses on the cask; `brew audit --cask --online --strict smartandpoint/tap/deepdrivecleaner` reports no problems when run on a machine with current Command Line Tools (on the reference machine it refuses to start with CLT 16.3 until the Command Line Tools for Xcode 27 are installed); a rerun of `scripts/publish-release.sh` after the publish goes through the resume path: it downloads the release's asset, matches its digest and ends with "already holds this cask — nothing to publish" (a cask-only fix takes the same path and differs only in the final write)
- [ ] On the reference machine: `which ddc` is `/opt/homebrew/bin/ddc`; `~/.cargo/bin/deepdrivecleaner` and `~/.cargo/bin/ddc` no longer exist; `ddc schedule install` writes `/opt/homebrew/bin/deepdrivecleaner` into the plist; after the Full Disk Access grant, `launchctl kickstart -k gui/$UID/com.smartandpoint.deepdrivecleaner` adds a `background.log` line with `inaccessible 0`, and `last-scan.json` has `full_disk_access: true`; after `brew reinstall --cask deepdrivecleaner` (same path) a second kickstart still logs `inaccessible 0` without a new grant
- [ ] README's install section leads with Homebrew and its Full Disk Access section names the Caskroom binary and the re-grant after `brew upgrade`, not `~/.cargo/bin`; `make check` passes

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- Prebuilt binary, not a source build: a source formula would need `depends_on "rust" => :build` (the toolchain is a large download, and `rust-version` is 1.89 while some machines have an older Homebrew Rust). The trade-off is that users run our build; the release notes name the commit, the toolchain, the build command and the checksum.
- `aarch64-apple-darwin` only: every test and measurement ran on Apple Silicon. The ADR "Rust core: library + CLI with JSON output" planned a universal build with `lipo`; an Intel build can follow as a second tarball when someone can test it.
- Signed with the SmartAndPoint Developer ID (hardened runtime; no entitlements are needed for `getattrlistbulk`, `setiopolicy_np`, `proc_pidinfo`/`sysctl` on the user's own processes, `NSFileManager` trashing or the `osascript`/`launchctl` subprocesses) and notarized, as GrammarHelper is. The designated requirement of a Developer ID signature is the identifier plus the team, so the `csreq` that TCC stores stays satisfied by every later build (an ad-hoc signature is matched by its cdhash and fails on every rebuild). TCC also keys a bare executable by its absolute path (`client_type = 1`; this machine's TCC database holds `~/.cargo/bin/ddc` and `~/.cargo/bin/deepdrivecleaner` as two clients), so the grant survives `brew reinstall` at the same Caskroom path but not `brew upgrade`. The caveats and the README say to remove the old entry and add the new binary after an upgrade. The lasting fix is the menu bar app (next epic): an app bundle is keyed by its bundle id.
- Notarization lets the binary run on other Macs although a cask download carries a quarantine attribute. A bare binary cannot be stapled, so Gatekeeper checks the ticket online.
- Rejected: a Homebrew `service do` block (`brew services`) would also give an upgrade-proof plist, but it would be a second agent with the same purpose, without the user's threshold and interval, and only for Homebrew installs.
- Released while "Safe cleanup: ddc clean" still waits for its live-run criterion (the owner's interactive clean on the reference machine); its automated criteria pass. The owner asked for the release in this order.
- A bad release is fixed by a version bump: the script never replaces a published version's tarball. A cask-only correction at the same version goes through the resume path, which takes the tarball from the release itself, so a `make dist` run after the release does not get in its way. The resume allowlist is deliberately narrow: any other change on `main` since the tag, documentation included, means the next cask goes out with a new version.
- When a check after `gh release create` fails (the uploaded digest, the tag on origin), the release already exists, and a rerun resumes with whatever it holds, checked again only for its digest, signature, team and version. Inspecting the release by hand is the gate there.
- The binary is notarized but not stapled, so its first run after install or upgrade needs the network for the ticket lookup.
- Follow-ups: `ddc schedule status` does not yet show whether the last background run had Full Disk Access, and the background run does not notify when the grant was lost after an upgrade; with `HOMEBREW_CASK_OPTS=--no-binaries` there is no `/opt/homebrew/bin` link and the agent records the versioned Caskroom path.

## Dependencies

- [[epics/DDC-001/stories/story-scheduled-background-scan|Scheduled background scan]]
- [[adr/rust-core-library-and-cli-with-json-output|Rust core: library + CLI with JSON output]]

## Attachments

-

---

*Last updated: 2026-10-18*

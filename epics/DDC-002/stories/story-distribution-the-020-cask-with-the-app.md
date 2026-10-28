---
type: story
id: "story-distribution-the-020-cask-with-the-app"
epic: "DDC-002"
title: "Distribution: the 0.2.0 cask with the app"
status: in-progress
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-20
external_refs: {design_canvas: "https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV"}
tags: [release, homebrew, distribution, disclaimer]
code_refs: []
specs: []
started_at: "2026-10-20T10:50:33.966Z"
closed_at: null
plan_updated_at: "2026-10-20T11:04:49.221Z"
---

# Distribution: the 0.2.0 cask with the app

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

DeepDriveCleaner 0.2.0 ships as one cask, `deepdrivecleaner`, with the app and the CLI linked from inside the bundle (ADR "Menu bar app: Swift shell over the ddc CLI", §8). It is signed by the existing team, 8MSDJTBW2D. The 0.1.0 release tooling is extended to a zipped, stapled app, and the reference machine is upgraded from 0.1.0.

**Scope of 0.2.0 (the owner's decision, 2026-10-20): what is built now.** The menu bar popover, the Review and clean window and picking the user's own files ship. "Settings: folders, types, notifications" and "Background scan and notifications in the app" do not: they ship in 0.3.0. So in 0.2.0:
- the Settings window holds only Open at login;
- scheduled scans stay the CLI's (`ddc schedule install`, as in 0.1.0), and the app has no scheduler agent;
- after `brew upgrade` (which quits the app) the app opens again at the next login if Open at login is on (the login item is opt-in), or by hand: nothing reopens it sooner.

**Three additions from the owner (2026-10-20).**
- **A plain-language disclaimer next to the license.** `DISCLAIMER.md` beside `LICENSE.md`, and a Disclaimer section in the README: DeepDriveCleaner deletes files and folders, and can remove almost anything it is told to, by its user or by an agent acting for them. It is used at the user's own risk: nobody who made it, neither the owner nor any developer or contributor, is responsible for what it removes. It adds to the license's "No Liability" section and never narrows it; the license text itself is not changed.
- **A warning when `ddc` starts in a terminal.** Before any other output, the CLI writes one short warning to stderr, two lines: what it is (a cleaner that can delete almost anything) and that it is used at one's own risk, with the absolute link to `DISCLAIMER.md` (a link in a terminal is clickable, and the tarball's and the app's copies live in different places). Only when stderr is a terminal: the app, launchd, pipes and redirects never see it, so JSON output and scripts are unchanged. Every command gets it, `--help`, `help` and usage errors too; only `--version` does not. Its `warning:` label is colored unless `NO_COLOR` is set.
- **A detailed CLI guide in the README**: install, every command and its options with examples (`scan`, `clean` interactive / `--yes` / `--class` / `--permanent` / `--dry-run` / `--items`, `rules`, `schedule`; the hidden `notify` named as internal), the exit codes, where data and logs live, and how the CLI and the app share them.

## Decomposition

Rough until the plan gate.

- [ ] `make dist` builds both assets from one commit: the CLI tarball (as in 0.1.0, now with `DISCLAIMER.md`) and the zipped, stapled app (`scripts/build-app.sh --dist`, story "App skeleton and build pipeline"), which refuses a version mismatch between Cargo and the app's `MARKETING_VERSION`, and refuses unless `dist/BUILD_COMMIT` (written by `build-dist.sh`) is HEAD. `make app-dist` stays as an alias
- [ ] The tarball's CLI is signed as `com.smartandpoint.deepdrivecleaner.cli`, as the CLI inside the app is (ADR "Menu bar app: Swift shell over the ddc CLI", §4), so no executable but the app carries the app's bundle id; a manual install from the tarball needs its Full Disk Access grant once more, and the release notes say so
- [ ] Deferred to a later story: extracting the signing, notarization and retry helpers into `scripts/lib/release.sh`. Both scripts work and notarized 0.1.0; a refactor of them is not a step of this release
- [ ] `scripts/publish-release.sh`:
  - assets `DeepDriveCleaner-<version>.zip` (the cask's) and the CLI tarball (for a manual install, and for Macs before macOS 26, which the 0.2.0 cask does not reach);
  - checks the zip holds `DeepDriveCleaner.app` alone, the app's signature (`--deep --strict`), identifier, team, stapled ticket, version, bundle id and `LSMinimumSystemVersion` (26.0, the cask's `depends_on macos:`), the embedded CLI's identifier, team and version, and the tarball's listing (with `DISCLAIMER.md`) and binary as in 0.1.0, now with the `.cli` identifier. Nothing opens the app;
  - keeps the 0.1.0 resume path (the cask's checksum comes from the release's zip); the tap is written through `gh api`
- [ ] Cask template:
  - `app "DeepDriveCleaner.app"`;
  - two `binary` stanzas from `#{appdir}/DeepDriveCleaner.app/Contents/Helpers/deepdrivecleaner`, one of them `target: "ddc"`;
  - `depends_on macos: :tahoe` (as GrammarHelper's cask) and `depends_on arch: :arm64`;
  - `uninstall quit: "com.smartandpoint.deepdrivecleaner"`;
  - `zap` the data, logs and preferences, with `launchctl:` for `com.smartandpoint.deepdrivecleaner` (0.1.0) and `com.smartandpoint.deepdrivecleaner.scheduler` (named now, so a later zap needs no cask change);
  - caveats: Full Disk Access goes to the app; a CLI started by launchd (`ddc schedule install`) has a new path inside the app and needs its grant once more; the disclaimer, in three short lines with its link
- [ ] Version 0.2.0 in the workspace and the app; release notes `docs/releases/0.2.0.md` (what ships, what waits for 0.3.0, the disclaimer); README leads with the app, has the detailed CLI guide and the Disclaimer section, and says that `brew uninstall` without `--zap` can leave the app's entries in Login Items until macOS drops them
- [ ] `DISCLAIMER.md`, and the CLI's terminal warning with its tests. `LICENSE.md` and `DISCLAIMER.md` also go into the app's `Contents/Resources` (`scripts/build-app.sh`), since a cask install has no other copy; the publish script checks they match the repository's
- [ ] Release notes: a way back to 0.1.0 (uninstall the cask, install the 0.1.0 tarball by hand), and not to upgrade while the app is cleaning (`uninstall quit:` waits 10 s, a tool's own clean command can run longer)
- [ ] Reference machine: `brew upgrade --cask deepdrivecleaner` from 0.1.0 (the owner's)

## Implementation Plan

No covering spec; the design is the ADR "Menu bar app: Swift shell over the ddc CLI", §4 and §8. The planner's placement (2026-10-20) follows the 0.1.0 release's shape: shell scripts in `scripts/`, the template in `packaging/homebrew/`, notes in `docs/releases/`, one small CLI module wired from `main.rs`, black-box tests in `crates/ddc-cli/tests/`. No new package, no `scripts/lib/`.

1. **The warning.** `crates/ddc-cli/src/disclaimer.rs` holds the text and `warn_if_terminal()` (`std::io::stderr().is_terminal()`, color through `render::use_color` and `Env::no_color`). `main.rs` switches to `Cli::try_parse()` and warns before clap prints help or a usage error, unless the parse ended in `DisplayVersion`. Test `crates/ddc-cli/tests/disclaimer_cli.rs` runs ddc under `/usr/bin/script` with stdout redirected through `/bin/sh` to a file, so only stderr is on the terminal (the Ctrl-C test's mechanism).
2. **The disclaimer and the README.** `DISCLAIMER.md` beside `LICENSE.md`. README: app first, install (macOS 26, the tarball for older Macs, `--zap`, the Login Items note), the menu bar app, "The command line" with a "### `ddc <command>`" part per visible command, exit codes, where things live, settings; then risk classes, safety, Full Disk Access, the Disclaimer section. Test `crates/ddc-cli/tests/readme_cli.rs` parses `--help` of every visible command and checks the guide both ways (every option and possible value named; no option named that the command lacks), plus the Disclaimer section's link.
3. **Version 0.2.0.** `Cargo.toml` and `Cargo.lock`; `MARKETING_VERSION` 0.2.0 and `CURRENT_PROJECT_VERSION` 2 in both build configurations of the project. The 0.1.0 fixtures stay: they are history.
4. **Builds.** `scripts/build-dist.sh`: `DISCLAIMER.md` in the tarball and its listing check; the CLI identifier `.cli`. `scripts/build-app.sh --dist`: refuses unless `dist/BUILD_COMMIT` is HEAD. `Makefile`: `dist: check app-test` runs both scripts; `app-dist: dist`.
5. **Publish.** `scripts/publish-release.sh`: two assets (`DeepDriveCleaner-<v>.zip`, the cask's; the tarball), both on the new-release and the resume path, both digests checked; the zip's and the tarball's checks as in the decomposition; the notes block names both assets, both identifiers and both checksums. The cask's `sha256` is the zip's.
6. **Cask.** `packaging/homebrew/deepdrivecleaner.rb.tmpl` as in the decomposition, in the order the cask cops want (`depends_on` arch before macos, `app` and `binary` together, `uninstall`, `zap` with `launchctl` before `trash`, sorted arrays, `quit:` a string). Checked by `ruby -c` and `brew style` on a rendered copy.
7. **Release notes** `docs/releases/0.2.0.md`: what ships, what waits for 0.3.0 (with what "Scheduled scans off" and Pause mean now), the upgrade notes (Full Disk Access once to the app, the agent's new path, the `.cli` identifier, `brew upgrade` quits the app, macOS 26), and the disclaimer with an absolute link.
8. `make check`, `make app-test`; reviewer; commit both repositories and push (git via /sp, the owner approves).
9. **Release precondition:** every dependency story has passed its reviewer. What stays open in them ("App skeleton and build pipeline", "Menu bar popover", "Review and clean window", "Review window: sorted sidebar, Select all and folder groups", "User data: pick your own files for the Trash") is the owner's manual checks, which run on the 0.2.0 build the owner asked to release (2026-10-20); a check that fails there becomes a 0.2.1 fix, and its story stays open until then.
10. In the main thread: `make dist` (signing, notarization), `scripts/publish-release.sh --dry-run`, shown to the owner; the publish only on the owner's fresh approval. Then the owner's manual checks.

The public texts about Full Disk Access (README, release notes, caveats) say the grant *should* survive an upgrade until the reinstall checks below confirm it; the critic (2026-10-20) noted the GrammarHelper precedent of "App skeleton and build pipeline". The owner may run the remove-then-copy check before `make dist` on the reference machine (`make app`, the grant on it, `rm -rf` and `ditto` the bundle, open it, read its status).

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] `brew style` reports no offenses on the rendered cask; `brew audit --cask --online --strict` reports none on a machine with current Command Line Tools
- [ ] `scripts/publish-release.sh --dry-run` prints the rendered cask and notes; the 0.1.0 refusals still hold (dirty tree, HEAD not on `origin/main`, missing notes, stale build, an asset not signed by the configured team)
- [ ] Manual check on the reference machine: `brew upgrade --cask deepdrivecleaner` from 0.1.0 installs `/Applications/DeepDriveCleaner.app`, `ddc --version` prints 0.2.0, and `/opt/homebrew/bin/ddc` resolves into the bundle's `Contents/Helpers`. Full Disk Access is granted once, to the app
- [ ] Manual check on the reference machine, which runs the 0.1.0 agent of `ddc schedule install`: after the upgrade, with Full Disk Access on the app only, `launchctl kickstart gui/<uid>/com.smartandpoint.deepdrivecleaner` and read `full_disk_access` from `~/Library/Logs/DeepDriveCleaner/background.log`; then add `/Applications/DeepDriveCleaner.app/Contents/Helpers/deepdrivecleaner` as the caveats say and read it again. If the first run already has access, or the file picker adds the app instead of the path, the caveats, the README and the release notes are reworded from the result (the caveats are a cask-only fix: commit the template change and rerun the publish script, whose resume path re-renders the cask for the same release; only after that, commit the README change on `main` and edit the release's notes with `gh release edit`, since a README or notes commit before the resume run trips its guard)
- [ ] Rust test: under a pseudo-terminal a representative set of commands (no arguments, `--help`, `-h`, `help`, a usage error, the help of `scan`, `clean`, `schedule`, `schedule status` and `notify`, `rules`, `scan`, `clean --dry-run`) starts with the warning on stderr; the warning precedes dispatch in `main.rs`, so the commands the test cannot run safely (`schedule install|uninstall|status`, `notify ack|snooze`) get it the same way; and `--version` prints only the version; with stderr a pipe (as the app runs it) or a file (as launchd runs it) there is no warning. `scan --json` stdout is the saved report plus a newline both with stderr on a terminal and on a pipe, and the two reports are equal but for `generated_at`, `duration_ms` and the volumes' free space, which change between runs
- [ ] `DISCLAIMER.md` is in the repository beside `LICENSE.md`, in the CLI tarball, and linked from the README's Disclaimer section and from the release notes; the license text is unchanged
- [ ] The README's CLI guide names every subcommand and option that `ddc --help` and each `ddc <command> --help` print (checked by a test that compares them), with an example for each command
- [ ] Manual check: after `brew reinstall --cask deepdrivecleaner` (which quits the app), the app's Full Disk Access grant still holds when it is opened again
- [ ] Manual check: `brew uninstall --zap --cask deepdrivecleaner` leaves no `com.smartandpoint.deepdrivecleaner` agent: `launchctl print` finds no job
- [ ] Manual check on a second Mac or a fresh user account: the downloaded app opens without a Gatekeeper warning (`spctl` accepted, Notarized Developer ID)
- [ ] Clean-machine access check on the owner's work laptop after `brew install --cask deepdrivecleaner` (owner's decision, 2026-10-18; it replaces the spike of "App skeleton and build pipeline"). Terminal has no Full Disk Access there, and the grant goes to DeepDriveCleaner.app only. Read from the app's own status and `~/Library/Logs/DeepDriveCleaner/app.log`:
  - before the grant the app shows no Full Disk Access; after it, a Scan now reports `full_disk_access: true`;
  - after `brew reinstall --cask deepdrivecleaner`, which removes the app and copies the new one, the grant still holds.
  The scheduled-scan half of this check (a tick launching the app with `--scheduled`, the notification's name and icon) moves to "Background scan and notifications in the app", which builds the scheduler (0.3.0). If the grant does not survive the remove-then-copy, Rationale 1 and this cask are revised

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- A zip, not a DMG: the cask is the install path, and a zip of a stapled app also works when downloaded by hand.
- `uninstall quit:` runs on upgrade and reinstall too. In 0.2.0 nothing reopens the app before the next login (the scheduler agent comes with 0.3.0); the release notes say so, and the ADR's §8 carries a dated note.
- In 0.2.0 the popover's "Scheduled scans off" and the menu's Pause scans are about the app's own scans; an agent from `ddc schedule install` keeps its schedule. The release notes say so; the UI changes with the scheduler in 0.3.0.
- The disclaimer and the warning are not legal advice; a limitation of liability may not hold everywhere (consumer law). They state the risk plainly, which is their main job.
- The CLI's path becomes stable inside `/Applications`, so the grant of a CLI started by launchd (`ddc schedule install`, for people who use only the CLI) survives upgrades.
- GrammarHelper's cask writes `depends_on macos: :tahoe`; current Homebrew reads a bare symbol as that version or later, and `brew style` decides the accepted form.
- `settings.json` refuses unknown keys ("CLI: shared settings, types and the notify decision"), so a separately installed CLI older than the app that wrote the file exits 1 on every command until it is upgraded. The cask ships the app with its own CLI, so the two always match. 0.1.0 does not read `settings.json` at all, so this matters from 0.3.0 on, when the app's Settings write new keys; the 0.3.0 release notes say so.
- `CURRENT_PROJECT_VERSION` is bumped by hand (2 for 0.2.0) and no script checks it; `build-app.sh` checks only `MARKETING_VERSION`. LaunchServices does not depend on it; a forgotten bump is invisible but harmless.
- The team switch is not part of this release: a switch to an organization team is later work, tracked outside this vault.

## Dependencies

- [[epics/DDC-002/stories/story-app-skeleton-and-build-pipeline|App skeleton and build pipeline]]
- [[epics/DDC-002/stories/story-cli-clean-an-explicit-item-list|CLI: clean an explicit item list]]
- [[epics/DDC-002/stories/story-cli-shared-settings-types-and-the-notify-decision|CLI: shared settings, types and the notify decision]]
- [[epics/DDC-002/stories/story-menu-bar-popover|Menu bar popover]]
- [[epics/DDC-002/stories/story-review-and-clean-window|Review and clean window]]
- [[epics/DDC-002/stories/story-review-window-sorted-sidebar-select-all-and-folder-groups|Review window: sorted sidebar, Select all and folder groups]]
- [[epics/DDC-002/stories/story-user-data-pick-your-own-files-for-the-trash|User data: pick your own files for the Trash]]
- Not a dependency of 0.2.0 any more (the owner's decision, 2026-10-20): [[epics/DDC-002/stories/story-settings-folders-types-notifications|Settings: folders, types, notifications]] and [[epics/DDC-002/stories/story-background-scan-and-notifications-in-the-app|Background scan and notifications in the app]], both 0.3.0

## Attachments

- Design canvas: https://claude.ai/artifact/1mdoyc7asMbj2qbtkDG2PV

---

*Last updated: 2026-10-20*

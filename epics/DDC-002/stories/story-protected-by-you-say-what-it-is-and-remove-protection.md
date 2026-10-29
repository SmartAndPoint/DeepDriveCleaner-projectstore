---
type: story
id: "story-protected-by-you-say-what-it-is-and-remove-protection"
epic: "DDC-002"
title: "Protected by you: say what it is, and Remove Protection"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-29
updated: 2026-10-29
external_refs: {}
tags: [core, cli, app, review, user-data, keep-marker, adr-amendment, user-report]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Protected by you: say what it is, and Remove Protection

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

The owner, 2026-10-29: "Protected by you" listed folders with disabled checkboxes and no button; a user will not understand what it is, and it needs a delete like every other group. (The 74 folders on the screenshot came from a foreign report; story "Review window: say when the saved scan is not of the home folder" fixes that.) The analysis is in the research note "Removal wherever a size is shown: coding agents, protected folders and the whole disk" (Finding 2).

**Why there is no delete.** `user.kept` is what the user protected: a `.ddc-keep` marker in a folder or above it, an exclusion (in Settings or a `--exclude` flag), or a skip (since the 2026-10-28 amendment, a skipped folder is excluded). The safety ADR (§2, §5) never removes a kept folder: deleting it in one click would remove exactly what the user asked to keep. So the way out removes the **protection**, never the folder. After it, the next scan lists the folder in its own group, where it can be cleaned with that group's class and checks.

**The words.**

- The group's blurb: "Folders you protected with a .ddc-keep file, an exclusion or a skip. DeepDriveCleaner never removes them. Remove the protection to clean one."
- Each row's second line says how it is protected: "Protected by .ddc-keep in <folder>"; "Excluded in Settings" or "Excluded by a command-line flag" when the row's own folder is excluded; "Holds a folder skipped in Settings: <path>", "Holds a folder skipped by a command-line flag: <path>", "Holds an excluded folder: <path>" when the skipped or excluded folder lies inside the row's finding, whose path is not the skipped one. When the report knows it, it adds what the folder would be otherwise: "otherwise: node_modules".
- The User data section's shared blurb stops promising "pick files one by one" for groups where nothing can be picked.

**The report.** Today a kept finding carries only `Note::KeptByMarker` or `Note::Excluded` (`classify.rs`). It gains:

- what keeps it: each marker's folder, with its device and inode; or the exclusion (Settings or flag), with the excluded path when it lies inside the finding; or the skip, with the skipped path and its origin, Settings or flag (`ddc scan --skip` and `ddc clean --skip` exist, and `settings.rs` merges flag skips into the same list as Settings skips; the origin is kept today only for an error message);
- the rule that would have claimed the folder, when the walker knows it. A kept finding comes from three sites (`rules/visitor.rs`, `rules/classify.rs`):
  - `claim_unless_kept` (a known path or table rule with a marker at its top) and the marker check in `after_read` (which must still run the matchers): the displaced rule is in hand;
  - the re-tag in `classify.rs`, when an exclude or a skip lies inside a finding (`excluded_inside`): the finding's own rule is in hand. A skipped folder itself is never a finding: `before_read` returns `Pre::Skip` and never opens it; what the report shows is the finding that holds it;
  - the exclude branch of `before_read`, for an exclude at or above the folder: it returns `Pre::Claim(kept)` and the walk turns its hooks off, so `after_read`'s matchers never see the folder.
- **The limit.** Only in that last site is the rule lost: there the displaced rule is known for table rules (known paths, regions, children of a known path), not for matcher rules such as `node_modules`. The story keeps that limit; running the matchers on a claimed listing is a larger change and not part of it. Nothing beneath a marker was classified either.

The fields are optional; the scan schema stays at version 1.

**Remove Protection…** on a row:

- **A marker:** a new `ddc keep` subcommand, `ddc keep --remove <marker>`. It
  - takes the data-dir lock exclusively (a running clean re-checks markers as it acts, `exec/guard.rs`);
  - refuses a marker outside the current scan roots;
  - opens the marker's folder with `O_RDONLY|O_DIRECTORY|O_NOFOLLOW`, compares its device and inode with the report, and unlinks `.ddc-keep` with `unlinkat` only if `fstatat(AT_SYMLINK_NOFOLLOW)` says it is a regular file: the mirror of `KeepMarker.protect`. A symlink or a folder named `.ddc-keep` also protects (`rules/visitor.rs`); it is refused with its path, and the row says to remove it in Finder;
  - writes a journal line: the marker's path, the folder's identity, and the findings it kept;
  - answers `--json` with a `schema_version`, like the other contracts the app reads.

  The confirmation names the kept findings and the rule that would claim each, and says that what lies beneath appears at the next scan. It warns when git tracks the marker: someone else may have committed it, and removing it changes the checkout. The check runs only when a real git binary exists at one of four fixed paths, tried in order: `/Library/Developer/CommandLineTools/usr/bin/git`, `/Applications/Xcode.app/Contents/Developer/usr/bin/git`, `/opt/homebrew/bin/git`, `/usr/local/bin/git`. It never goes through `/usr/bin/git` or `xcrun`, whose shim shows an install dialog on a Mac without the tools, and it runs no `xcode-select`. An Xcode elsewhere is not found, and the check is then "not known". It is `git -C <folder> --no-optional-locks -c core.fsmonitor=false ls-files --error-unmatch .ddc-keep`, with a timeout. No git, or a failure, means "not known", and no warning.

  After the command the app runs Scan Now. Until it ends, the row shows "Protection removed. Scanning…".
- **A Settings exclusion, or a finding holding a skip or a Settings exclusion:** the same edit as Settings → Folders → Remove (`FolderList.removeExclusion` removes a path from both lists), after a confirmation; then Scan Now. The report's path is canonical (skips in `settings.rs`, excludes in `report/mod.rs`) and `removeExclusion` matches the Settings entry by exact string, so the edit removes the Settings entry whose `realpath(3)` equals the reported path.
- **A command-line exclusion or skip:** no button; the row says "The next scan without the flag lists this folder."
- **Off** for a report whose roots lie outside the app's: the app disables it, as story "Review window: say when the saved scan is not of the home folder" disables Clean and Protect.

**Before it starts (the work-start gate).**

- A safety ADR amendment: §2, removing a marker through `ddc keep --remove` with the checks above; §7, the journal's new `kind`, with a schema version bump if its versioning policy asks for one.
- A menu bar ADR amendment: §7, `ddc keep --remove` is a destructive command with one caller (the Remove Protection confirmation), enforced by a contract test like `ContractTests.agentsCallers`; §6, its `--json` as a new contract.
- The owner's answer to the research note's open question 1: two steps (protection gone, rescan, then clean in the folder's own group), as written here, or one flow that also cleans under the underlying rule.

## Decomposition

- [ ] ADR amendments (safety §2, §7; menu bar §6, §7), critic-reviewed.
- [ ] Core: the kept finding's new fields from the three sites (`visitor.rs`, `classify.rs`), with the skip's origin; schema and goldens.
- [ ] Core/CLI: `ddc keep --remove` with the lock, the roots check, the identity check, the regular-file unlink, the journal line; `--json`.
- [ ] DDCKit: decode the fields; the Settings entry whose `realpath(3)` equals the reported path, then `removeExclusion` with the entry as written; the row wording; the blurb; the confirmation text with the git warning; the git binary lookup by path.
- [ ] App: Remove Protection… in the row's context menu and as a row button, the confirmation, Scan Now after, the row while it scans; the Settings route; the contract test.

## Implementation Plan

## Acceptance Criteria

- [ ] Test (core): in a temp HOME, a `node_modules` folder with a `.ddc-keep` at its top is a kept finding whose report names the marker's folder (with device and inode) and the rule `node.node_modules`; a marker deeper inside a claim names its own folder; a folder excluded in Settings names the exclusion, one excluded by `--exclude` the flag; a finding holding a folder skipped in Settings names the skip's path, the origin Settings and its own rule; one holding a `--skip` folder names the origin flag; a finding holding an excluded folder names that path and its own rule; a known-path folder that is excluded names its rule.
- [ ] Test (CLI): `keep --remove` removes a regular-file marker and writes the journal line; refuses a marker whose folder's identity changed, a symlink marker, a folder marker, a marker outside the roots, and a run while a clean holds the lock.
- [ ] Test (CLI): after `keep --remove`, a scan lists the folder in its own group.
- [ ] Test (DDCKit): the row wording for each kind of protection (marker, Settings exclusion, flag exclusion, a held Settings skip, a held flag skip, a held exclusion), with and without the underlying rule; the blurb; the git warning when the check says tracked, none when it fails, says untracked, or no git binary exists (and then nothing is executed).
- [ ] Test (DDCKit): the Settings edit removes an entry written through a symlink whose `realpath(3)` equals the reported path.
- [ ] Test (DDCKit): the contract test finds exactly one caller of `keep --remove`.
- [ ] Manual (owner): protect a folder from the app, then Remove Protection: the confirmation names it, the app rescans, the folder is in its own group and can be cleaned. On a finding holding a skipped folder, Remove Protection removes the skip from Settings → Folders.

## Final Summary

## Technical Notes

- Tests make markers in a temp HOME only. No test touches a real `.ddc-keep`, and no test runs a real git: a script stands in for it.
- The git check runs only on a marker inside a folder with a `.git` above it, within the scan root.
- `remove_` is a forbidden substring in `free/` (`structure.rs`); the new command lives outside `free/`.

## Dependencies

- Research note "Removal wherever a size is shown: coding agents, protected folders and the whole disk" (Finding 2, open question 1)
- "Review window: say when the saved scan is not of the home folder" (DDC-002): the foreign-report rule; this story waits for it to close
- Safety ADR "Cleanup safety model: risk classes, plan and confirmation" (§2, §5, §7) and ADR "Menu bar app: Swift shell over the ddc CLI" (§6, §7): the amendments this story adds

## Attachments

-

---

*Last updated: 2026-10-29*

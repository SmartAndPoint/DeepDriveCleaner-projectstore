---
type: story
id: "story-protected-by-you-say-what-it-is-and-remove-protection"
epic: "DDC-002"
title: "Protected by you: say what it is, and Remove Protection"
status: in-progress
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-29
updated: 2026-10-30
external_refs: {}
tags: [core, cli, app, review, user-data, keep-marker, adr-amendment, user-report]
code_refs: []
specs: []
started_at: "2026-10-30T14:44:22.789Z"
closed_at: null
plan_updated_at: "2026-10-30T14:44:22.789Z"
---

# Protected by you: say what it is, and Remove Protection

| Field | Value |
|---|---|
| **Epic** | [DDC-002](../epic.md) |
| **Status** | in-progress |
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

The fields are one optional `protection` object on a kept finding; the scan schema stays at version 2.

**Remove Protection…** on a row. It is offered only when the row has exactly one removable protection: one marker, or one Settings exclusion or skip. A row with several lists them all in its second line and says to remove them in Finder or in Settings → Folders.

**Remove Protection…**:

- **A marker:** a new `ddc keep` subcommand, `ddc keep remove <folder> --folder-dev <dev> --folder-ino <ino> --json`. It
  - without the identity, asks at a terminal before any lock, and refuses without one;
  - takes the data-dir lock exclusively (a running clean re-checks markers as it acts, `exec/guard.rs`);
  - refuses a folder outside the scan roots: the Settings roots, as `disk free` resolves them. The saved report supplies only the rule the marker kept;
  - opens the marker's folder with `O_RDONLY|O_DIRECTORY|O_NOFOLLOW`, compares its device and inode with the report, and unlinks `.ddc-keep` with `unlinkat` only if `fstatat(AT_SYMLINK_NOFOLLOW)` says it is a regular file: the mirror of `KeepMarker.protect`. A symlink named `.ddc-keep` also protects, and so does a folder of that name at the top of a finding (`rules/visitor.rs`; deeper inside a finding the walk records only non-folder markers, which this story leaves as it is); either is refused with its path, and the row says to remove it in Finder;
  - writes a journal line for every decision under the lock, refusals included: the folder, its identity, the rule it kept, and the result;
  - answers `--json` with a `schema_version`, like the other contracts the app reads (menu bar ADR §6.12), and exits as `disk free` does: 0 once the attempt was made, whatever its result. A folder with no marker any more is `absent`, and the app rescans as after a removal.

  The confirmation names the kept findings and the rule that would claim each, and says that what lies beneath appears at the next scan in its own group, where it may be selected for cleanup by default. It warns when git tracks the marker: someone else may have committed it, and removing it changes the checkout. The check runs only when a real git binary exists at one of four fixed paths, tried in order: `/Library/Developer/CommandLineTools/usr/bin/git`, `/Applications/Xcode.app/Contents/Developer/usr/bin/git`, `/opt/homebrew/bin/git`, `/usr/local/bin/git`. It never goes through `/usr/bin/git` or `xcrun`, whose shim shows an install dialog on a Mac without the tools, and it runs no `xcode-select`. An Xcode elsewhere is not found, and the check is then "not known". It is `git -C <folder> --no-optional-locks -c core.fsmonitor=false ls-files --error-unmatch .ddc-keep`, with a timeout. No git, or a failure, means "not known", and no warning.

  After the command the app runs Scan Now. Until it ends, the row shows "Protection removed. Scanning…".
- **A Settings exclusion, or a finding holding a skip or a Settings exclusion:** the same edit as Settings → Folders → Remove (`FolderList.removeExclusion` removes a path from both lists), after a confirmation; then Scan Now. The report's path is canonical (skips in `settings.rs`, excludes in `report/mod.rs`) and `removeExclusion` matches the Settings entry by exact string, so the edit removes the Settings entry whose `realpath(3)` equals the reported path.
- **A command-line exclusion or skip:** no button; the row says "The next scan without the flag lists this folder."
- **Off** for a report whose roots lie outside the app's: the app disables it, as story "Review window: say when the saved scan is not of the home folder" disables Clean and Protect.

**Before it starts (the work-start gate).**

- A safety ADR amendment: §2, removing a marker through `ddc keep remove` with the checks above; §7, the journal's new `kind`, with a schema version bump if its versioning policy asks for one.
- A menu bar ADR amendment: §7, `ddc keep remove` is a destructive command with one caller (the Remove Protection confirmation), enforced by a contract test like `ContractTests.agentsCallers`; §6, its `--json` as a new contract.
- The owner's answer to the research note's open question 1: two steps (protection gone, rescan, then clean in the folder's own group), as written here, or one flow that also cleans under the underlying rule. **Answered 2026-10-30: two steps.**

## Decomposition

- [ ] ADR amendments (safety §2, §7; menu bar §6, §7), critic-reviewed.
- [ ] Core: the kept finding's new fields from the three sites (`visitor.rs`, `classify.rs`), with the skip's origin; schema and goldens.
- [ ] Core/CLI: `ddc keep remove` with the lock, the roots check, the identity check, the regular-file unlink, the journal line; `--json`.
- [ ] DDCKit: decode the fields; the Settings entry whose `realpath(3)` equals the reported path, then `removeExclusion` with the entry as written; the row wording; the blurb; the confirmation text with the git warning; the git binary lookup by path.
- [ ] App: Remove Protection… in the row's context menu and as a row button, the confirmation, Scan Now after, the row while it scans; the Settings route; the contract test.

## Implementation Plan

Planner, 2026-10-30, against origin/main at eea7d3c. No covering spec. Four PRs, merged in order; each passes the Gate and can ship on its own.

1. **The report says what protects a finding** (core, schema, goldens; no new command, no UI).
   - `model.rs`: `Finding.protection: Option<Protection>` with `markers` (folder, dev, ino), `exclusions` (path, kind `exclude`/`skip`, origin `settings`/`flag`, `inside`) and `rule`. It is set only on `user.kept` findings. `Note` is unchanged, since a new variant would force a schema bump.
   - `settings.rs`: `resolve` keeps the excludes' origin and which skips came only from `--skip`. They pass through `ScanOptions`.
   - `rules/visitor.rs`: records the displaced rule at the three sites. `after_read` runs the matchers before it claims `kept`, but a marker still always claims `kept`. The exclude branch of `before_read` looks up table rules only.
   - `rules/classify.rs`: carries the protection and assigns deep markers by a sorted partition. A deep marker's folder identity is the `(dev, ino)` the walker holds at the `on_file` call (`here`), recorded with the marker, not read again by path: `on_file` gains the directory's identity as a parameter, and its five implementors take it.
   - Regenerate `docs/scan.schema.json` and the goldens. Add a DDCKit fixture finding with `protection` to prove old Swift still decodes the report.
2. **`ddc keep remove`** (core, CLI, journal, platform).
   - `DirFd::unlink_child` beside `rename_child` in `platform/macos/fsops.rs`, with a stub in `unported.rs` and a pin in `platform/mod.rs`.
   - `keep.rs`: `unprotect` checks, in order, the roots, the `O_NOFOLLOW` open, the identity, `stat_child` (a regular file only), then the unlink. `KEEP_SCHEMA_VERSION` 1, with `docs/keep.schema.json`.
   - `exec/journal.rs`: `JournalKind::KeepRemove`, `JOURNAL_SCHEMA_VERSION` 3 → 4; `tests/disk_free_cli.rs` and `ddc-core/tests/disk_free.rs` move their version pins to 4.
   - `cmd/keep.rs` follows `cmd/disk_free.rs`: the terminal question before the lock, then the exclusive lock, then `last-scan.json` (for the rule only) and the identity check. `clippy.toml`'s `remove_file` reason names `unlink_child` beside the executor.
   - A structure test: the keep code names no `remove_dir`, `remove_file`, trash, `Command::new` or executor, and `unlink_child(KEEP_MARKER)` appears once.
   - Update the README, `gate.rs`'s `EVERY` and `unported_cli.rs`.
3. **DDCKit**.
   - Decode `protection`, its enums strictly like `Note`: a new kind or origin bumps the scan schema.
   - The row wording and the blurb move into `CleanWording`; `note(.excluded)` follows the origin.
   - `RemoveProtection` builds the call with the identity and `--json`, through a queued `CLIRunner` kind.
   - `FolderList`: the Settings entry whose realpath equals the reported path. `removeExclusion` is called with the entry as written.
   - `GitTracking`: four fixed paths, an injectable runner, a timeout of about 2 s. It never runs `/usr/bin/git` or `xcrun`.
   - `ReviewModel.canRemoveProtection` and `removeProtection`; the rescan runs through the `Rescan` path, as after a clean.
4. **App**.
   - `ItemRow`: Remove Protection… in the context menu and as a button, with its confirmation and the row while it scans.
   - `AppDelegate` wires the Settings edit through `SettingsStore`, which stays the only writer.
   - `ContractTests.keepCallers` and `keepContracts`; `schemaEnums` gains the new enums.

Decisions taken at this gate: the owner chose two steps. Every decision under the lock is journaled. Without the folder's identity, the CLI asks at a terminal before the lock and refuses elsewhere. The exit codes follow `disk free`: a refusal under the lock or a failed unlink is in the report with exit 0 (exit 1 without `--json`). A row with several removable protections gets no button. The command is `ddc keep remove`, a subcommand. Critic round 1 (2026-10-30) gave the exit codes, the roots source, the prompt order, `absent`, strict decoding and the tests below.

## Acceptance Criteria

- [ ] Test (core): in a temp HOME, a `node_modules` folder with a `.ddc-keep` at its top is a kept finding whose report names the marker's folder (with device and inode) and the rule `node.node_modules`; a marker deeper inside a claim names its own folder; a folder excluded in Settings names the exclusion, one excluded by `--exclude` the flag; a finding holding a folder skipped in Settings names the skip's path, the origin Settings and its own rule; one holding a `--skip` folder names the origin flag; a finding holding an excluded folder names that path and its own rule; a known-path folder that is excluded names its rule.
- [ ] Test (CLI): `keep remove` removes a regular-file marker and writes the journal line; refuses a marker whose folder's identity changed, a symlink marker, a folder marker, a marker outside the roots, and a run while a clean holds the lock.
- [ ] Test (CLI): after `keep remove`, a scan lists the folder in its own group.
- [ ] Test (CLI): every refusal under the lock (`changed-since-scan`, `no-folder`, `not-a-file`, `outside-scan-roots`) writes a `keep-remove` journal line with that reason and exits 0 with `--json`; a folder with no marker is `absent`, journaled, exit 0; a held lock exits 75 and writes none; without `--folder-dev`/`--folder-ino` and without a terminal it exits 2 before the lock; the journal lines carry `schema_version` 4.
- [ ] Test (CLI): the `--json` report validates against `docs/keep.schema.json`; a structure test finds `unlink_child` called once, with `KEEP_MARKER`, and no removal, trash or executor call in the keep code.
- [ ] Test (DDCKit): the row wording for each kind of protection (marker, Settings exclusion, flag exclusion, a held Settings skip, a held flag skip, a held exclusion), with and without the underlying rule; the blurb; the git warning when the check says tracked, none when it fails, says untracked, or no git binary exists (and then nothing is executed).
- [ ] Test (DDCKit): the Settings edit removes an entry written through a symlink whose `realpath(3)` equals the reported path.
- [ ] Test (DDCKit): the contract test finds exactly one caller of `keep remove`; `keepContracts` validates the fixture against `docs/keep.schema.json`; `schemaEnums` covers the new enums.
- [ ] Test (DDCKit): `canRemoveProtection` is false for a row with two protections, for a flag-origin row, and when the report's scope is outside the app's roots; the `scan-report.json` fixture with `protection` decodes; the app rescans on `removed` and `absent` and not on `refused` or `failed`.
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

*Last updated: 2026-10-30*

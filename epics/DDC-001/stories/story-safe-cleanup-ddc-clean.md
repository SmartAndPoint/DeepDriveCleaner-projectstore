---
type: story
id: "story-safe-cleanup-ddc-clean"
epic: "DDC-001"
title: "Safe cleanup: ddc clean"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-17
updated: 2026-10-18
external_refs: {}
tags: [cli, cleanup, safety]
code_refs: ["crates/ddc-core/src/plan.rs", "crates/ddc-core/src/exec/", "crates/ddc-core/src/platform/macos/procs.rs", "crates/ddc-core/src/platform/macos/fsops.rs", "crates/ddc-core/src/rules/mod.rs", "crates/ddc-core/src/rules/catalog.rs", "crates/ddc-core/tests/clean_safety.rs", "crates/ddc-cli/src/cmd/clean.rs", "crates/ddc-cli/src/prompt.rs", "crates/ddc-cli/tests/clean_cli.rs", "bench/make_baseline.py"]
specs: []
started_at: "2026-10-17T10:16:47.568Z"
closed_at: "2026-10-18T06:36:35.298Z"
plan_updated_at: "2026-10-17T10:16:47.568Z"
---

# Safe cleanup: ddc clean

| Field | Value |
|---|---|
| **Epic** | [DDC-001](../epic.md) |
| **Status** | done |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

Removing the selected items under the rules of [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]: fresh scan → plan → selection → confirmation → execution with a re-check of every path and a journal.

## Decomposition

- [x] `ddc clean` always rescans under an exclusive lock; a plan with selection by class, lock file and activity (`--min-age`, default 14 days); `user-data` does not enter the plan; the scan root does not enter the plan
- [x] Filters: repeatable `--class` (without `--yes`, all classes except `user-data` are shown by default), `--category`, `--min-size`; `--dry-run` prints the plan and exits, no executor is created
- [x] Interactive selection by group (multi-select); a separate confirmation for `review`; a warning that the Trash does not free space
- [x] `--yes`: only items selected by default from the classes in `--class` (default `safe`); `review` in `--class` together with `--yes` is an error; without a TTY, `--yes` or `--dry-run` is required
- [x] Executor per method: `native` (absolute path, no shell, stdin closed, output to a journal file, timeout) → `delete` (rename to a tombstone, removal without following symlinks, on `EACCES` — `u+w` and retry) → `trash` (`NsFileManager`; for all `review`, including `rebuildable` downgraded to `review`; `--permanent` replaces it with `delete`) → `native-only` (only the command while the daemon is running; a command failure does not fall through to deletion)
- [x] Re-check: exists, is a directory, not a symlink, `(dev, ino)` of the finding and its parent match, `Catalog::verify` passes, inside the scan root and not equal to it, not a protected folder (by path and by `(dev, ino)`), not a project root, no `.git` at the top level, no `.ddc-keep` in the finding or its ancestors; tombstone via `renameat` from the verified parent descriptor
- [x] `in-use` check: the path from `KERN_PROCARGS2` (before symlink resolution), arguments, `VIRTUAL_ENV`/`CONDA_PREFIX`, the working directory (`PROC_PIDVNODEPATHINFO`) of any user process inside the finding; relative paths are joined with the process's working directory; owner-process check
- [x] All side effects go through traits (`CommandRunner`, `Trasher`, `ProcessProbe`, `Journal`) with fakes in tests; the executor refuses to run as root
- [x] Warning about local Time Machine snapshots
- [x] Journal `<log-dir>/actions.jsonl` with `schema_version`; summary: deleted / in Trash / freed per `statfs`
- [x] Owner applications by app bundle, a finding of the first real run (2026-10-18). The check was `p.name.starts_with(owner)`: the system's `CursorUIViewService` (`/System/Library/PrivateFrameworks/TextInputUIMacHelper.framework/…`) matched `Cursor` and skipped 15 caches of VS Code, Cursor and Windsurf (2.7 GiB) while none of them ran; `Electron` matches every Electron app; and one owner list for three editors let any one of them block all three. Now a process belongs to an app when its executable path lies under `Contents/MacOS/` or `Contents/Frameworks/` of the outermost `.app` bundle in that path (every app keeps its main binary in `Contents/MacOS`, Electron and Chrome helpers live under `Contents/Frameworks`; tools under `Contents/Developer`, such as Xcode's `clangd`, are not the app). An owner names that bundle: equal to the bundle name, or that name followed by a character that is not a letter or digit (`Adobe Photoshop 2025`, `Xcode-beta`, `PyCharm CE`). Inside a bundle the executable's own name is matched the same way, so a renamed bundle is still recognised (`Chrome.app` runs `Google Chrome`, `Xcode16.app` runs `Xcode`), as the old check recognised it. A relative executable path is joined with the process's working directory; a process with no resolvable absolute path matches no owner. The skip reason names the app's bundle, not its executable ("Cursor is running", not "Electron" or "Cursor Helper (GPU)"). `ide.editor-cache` is split into `ide.vscode-cache`, `ide.cursor-cache` and `ide.windsurf-cache`, each with its own owner; owners become bundle names (`Visual Studio Code`, `Cursor`, `Windsurf`, `Xcode`, `Google Chrome`, `Brave Browser`, `Firefox`, `Spotify`, `Slack`, `Adobe Lightroom`, `Adobe Photoshop`, and the JetBrains IDEs: `IntelliJ IDEA`, `PyCharm`, `WebStorm`, `GoLand`, `CLion`, `RustRover`, `DataGrip`, `Rider`, `PhpStorm`, `RubyMine`, `DataSpell`, `Fleet`)

## Implementation Plan

Route per the `projectstore:planner` plan (section "Safe cleanup") and ADR "Cleanup safety model: risk classes, plan and confirmation" §1, §5–§7 — this is the story's normative contract.

1. **Parent identity.** The scanner records the `(dev, ino)` of each finding's parent (`OwnerResult`/`Finding.parent_dev/parent_ino`, additive to schema v1) — for the re-check "the finding and its parent have not been swapped".
2. **`plan.rs`** — `build_plan(report, catalog, opts) -> Plan`: the plan excludes `user-data`, rules with method `none`, and the scan root itself; the action comes from the rule's method (`--permanent` turns `trash` into `delete`); selection comes from `Finding.selected`; filters `--class` (repeatable), `--category`, `--min-size`.
3. **`sys/procs.rs`** — processes of the current user: `proc_listallpids` → executable path and arguments from `sysctl(KERN_PROCARGS2)` (path before symlink resolution), `VIRTUAL_ENV`/`CONDA_PREFIX` from the environment, working directory from `proc_pidinfo(PROC_PIDVNODEPATHINFO)`, name for the owner-process check. **`sys/fsops.rs`** — `open(O_DIRECTORY|O_NOFOLLOW)` of the parent, `fstat`, `fstatat(AT_SYMLINK_NOFOLLOW)` of the finding, `renameat` into the tombstone from the same descriptor.
4. **`exec/guard.rs`** — re-check per ADR §5: the path exists, is a directory, not a symlink; the parent is a canonical path without symlinks, the `(dev, ino)` of the parent and of the finding match the scan; `Catalog::verify`; inside the root and not the root; not a protected folder (by case-insensitive path and by `(dev, ino)`); not a project root, no `.git` at the top; no `.ddc-keep` in ancestors or inside (one parallel walk of all selected findings by the scanner); `in-use` (absolute and relative paths, taking cwd into account, `VIRTUAL_ENV`/`CONDA_PREFIX`; with unknown cwd — block findings with the same name); the owner process is not running.
5. **`exec/mod.rs`** — `Executor` with ports `CommandRunner`, `Trasher`, `ProcessTable`, `Journal` (real implementations and fakes in tests); `Consent::{Interactive, AssumeYes{classes}}` — with `AssumeYes`, only items selected by default from the allowed classes, never `review`/`native-only`; refuses to run as root; `statfs` before/after; warning about local Time Machine snapshots (`tmutil listlocalsnapshots /`).
6. **`exec/delete.rs`** — tombstone via `renameat`, `remove_dir_all` without following symlinks, on `EACCES`/`EPERM` — `u+w` on directories inside the tombstone and retry. **`native`** — the command by absolute path without a shell, stdin closed, output to the journal, timeout 10 minutes; `location` check (`uv cache dir`, `go env …`, `pyenv root`, `rustup show home`) — otherwise the fallback; for `native-only`, a command failure does not fall through to deletion. **`trash`** — crate `trash` with `DeleteMethod::NsFileManager`.
7. **`exec/journal.rs`** — `<log-dir>/actions.jsonl`, one line per action with `schema_version`.
8. **CLI `ddc clean`** — exclusive lock, fresh scan, plan by group (class · rule), `--dry-run` (no executor is created), `--yes` (error with `--class review`), without a TTY `--yes` or `--dry-run` is required; interactive: toggle groups by number, a separate confirmation for each `review`, Trash warning, final confirmation; summary: deleted / in Trash / skipped with reasons / freed per `statfs`.
9. **Tests** — `crates/ddc-core/tests/clean_safety.rs` (all acceptance criteria at the executor API level with fakes), `crates/ddc-cli/tests/clean_cli.rs` (`--dry-run` does not change the tree and does not write the journal; `--yes` with an empty `PATH` and temporary `HOME`/`--data-dir`; lock already held).

10. **Owner applications by app bundle** (added after the first real run). `sys/procs.rs`: `app_bundle(exe)` (the outermost `*.app` component, only when followed by `Contents/MacOS` or `Contents/Frameworks`) and `ProcInfo::app_bundle()` (a relative exe joined with the cwd). `rules/mod.rs`: `OwnerApp` (was `OwnerProcess`, a name prefix) and `owns(owner, bundle)`, next to the catalog data they describe. `exec/guard.rs`: for every process inside a bundle, its bundle name and executable name computed once; the re-check matches either with `owns` and reports the bundle name in `OwnerRunning`. `rules/catalog.rs`: owners as bundle names, `ide.editor-cache` split per editor. `bench/make_baseline.py` and the local baseline TSV map the editor folders to the new ids. The existing owner test (`running_owner_process_skips_its_cache`) gets an executable path.

Files touched: `crates/ddc-core/src/{plan.rs,exec/,sys/procs.rs,sys/fsops.rs,scan/,model.rs,report/mod.rs,rules/mod.rs,rules/catalog.rs}`, `bench/make_baseline.py`, `crates/ddc-core/tests/clean_safety.rs`, `crates/ddc-cli/src/{cmd/clean.rs,prompt.rs,cli.rs}`, `crates/ddc-cli/tests/clean_cli.rs`, `Cargo.toml` (`trash`).

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [x] Test: a path replaced by a symlink between scan and cleanup is not deleted; the reason is in the journal — evidence: `crates/ddc-core/tests/clean_safety.rs` `symlink_swapped_in_after_scan_is_not_followed` (asserts the journaled `symlink` reason), `a_parent_replaced_by_a_symlink_is_refused`
- [x] Test: a directory with the same path and marker but a different inode is not deleted (`changed-since-scan`) — evidence: `clean_safety.rs` `same_path_and_marker_but_another_directory_is_not_deleted`
- [x] Test at the executor API level: `home`, the scan root, `home/Documents` and its variant with different letter case, and a project root are rejected even if passed into the plan directly — evidence: `clean_safety.rs` `protected_directories_and_project_roots_are_refused_even_if_planned`, `a_known_path_that_became_a_git_work_tree_is_refused`
- [x] Test: `--yes` deletes only selected `safe`; with `--class safe --class rebuildable` — also selected `rebuildable`; `--yes --class review` is an error; `review` is not deleted without interaction — evidence: `crates/ddc-cli/tests/clean_cli.rs` `yes_cleans_only_safe_items_by_default`, `yes_with_rebuildable_class_cleans_idle_projects`, `yes_with_review_class_is_rejected`; `clean_safety.rs` `assume_yes_never_touches_review_and_rejects_review_class`, `assume_yes_skips_active_projects_even_if_selected`
- [x] Test: the executor refuses to apply `delete`/`trash` to a `native-only` rule and any action to `user-data`, even if they are slipped into the plan (`action-not-allowed`); a failed native command does not lead to deletion — evidence: `clean_safety.rs` `native_only_rules_never_delete_or_trash` (a failing `docker` never falls back to deletion), `the_executor_refuses_actions_the_rule_does_not_allow`, `user_data_and_report_only_rules_never_enter_a_plan`
- [x] Test: a venv without a lock file goes to the Trash, and with `--permanent` it is deleted (executor, not only the plan); a directory swapped during the tool probe is not deleted; a partially unreadable finding is not deleted — evidence: `clean_safety.rs` `review_items_default_to_the_trash_unless_permanent`, `interactive_review_goes_to_the_trash`, `a_directory_swapped_while_a_tool_runs_is_not_deleted`, `partly_unreadable_findings_are_not_deleted`
- [x] Test: `--dry-run` does not change the tree (paths, types, inodes match before/after) and does not write the journal — evidence: `clean_cli.rs` `dry_run_changes_nothing_and_writes_no_journal`
- [x] Test: a tree with `0555` directories is deleted; an interrupted deletion leaves a tombstone that the next scan finds as `safe` — evidence: `clean_safety.rs` `read_only_trees_are_deleted`, `interrupted_deletion_leaves_a_tombstone_found_as_safe`
- [x] Test: a process launched via `<tmp>/.venv/bin/sleep` (a symlink to `/bin/sleep`) blocks deletion (`in-use`); the same for launching `./.venv/bin/sleep` with working directory `<tmp>` and for a working directory inside the finding — evidence: `clean_safety.rs` `processes_using_a_finding_block_it` (real processes), `a_relative_executable_with_unknown_cwd_blocks_same_named_findings`
- [x] Test: a `.ddc-keep` created in a finding after the scan cancels its deletion — evidence: `clean_safety.rs` `keep_marker_created_after_the_scan_cancels_deletion`
- [x] Test: each action produces one journal line with path, rule, class, method, sizes, result — evidence: `clean_safety.rs` `deletes_selected_items_and_journals_each`; live: 2007 lines in `~/Library/Logs/DeepDriveCleaner/actions.jsonl` for the 2007 items of the 2026-10-18 run
- [x] Test: `clean` does not start while the exclusive lock is held — evidence: `clean_cli.rs` `clean_does_not_start_while_the_lock_is_held`
- [x] Test: an app bundle is found for a main binary, an Electron or Chrome helper under `Contents/Frameworks` (Chrome's nested framework helper included), Firefox's nested `plugin-container.app`, an Adobe bundle in its versioned folder and a translocated app, and not for a tool under `Xcode.app/Contents/Developer`, `CursorUIViewService` (an `.xpc`), a helper under `Contents/Helpers` or `/usr/bin`; a relative executable resolves only with a working directory, and a bare process name gives no bundle — evidence: `crates/ddc-core/src/sys/procs.rs` `app_bundle_is_the_outermost_app_running_from_macos_or_frameworks`, `a_relative_executable_needs_the_working_directory`
- [x] Test at the executor level: `CursorUIViewService`, Xcode's `clangd`, a `CursorFoo.app` and a process named `Firefox` without an executable path hold back nothing; a running Cursor helper skips Cursor's caches and not VS Code's; `Adobe Photoshop 2025.app` holds back the Camera Raw cache; a renamed `Chrome.app` (a Chrome helper) and `Xcode16.app` hold back their caches by executable name; the skip reason names the bundle — evidence: `clean_safety.rs` `an_owner_is_an_app_bundle_not_a_process_name`, `a_running_editor_holds_back_only_its_own_caches`, `a_renamed_bundle_is_recognised_by_its_executable`, `running_owner_process_skips_its_cache`
- [x] Test on the live process table (macOS): no process whose executable lies under `/System/` (not `/System/Volumes/`), `/usr/` or `/Library/Apple/` matches any owner in the catalog by its bundle or executable name; it says so when no such process is readable or none runs from a bundle — evidence: `clean_safety.rs` `no_system_process_owns_a_cache`, run unskipped on the reference machine (36 system processes run from an app bundle; the reviewer's port checked all 311 system `.app` bundles: no collision)
- [x] Manual check on the reference machine (the baseline is local data): the baseline coverage check (`bench/check_baseline.py`) finds the editor caches under the three new ids — evidence: `deepdrivecleaner du --rules ~` then `python3 -I bench/check_baseline.py bench/baseline-2026-10-17.tsv <findings>`: covered 209.9 of 209.9 GiB = 100.0%, the 15 editor caches under `ide.vscode-cache`, `ide.cursor-cache`, `ide.windsurf-cache` (2026-10-18); its one miss is a `node_modules` inside `ZCode Computer Use.app`, skipped by design
- [x] Manual check on the reference machine: cleaning `safe` frees space, `df` confirms — evidence: the owner's `ddc clean` on 2026-10-18 (2007 items, 100.7 GiB selected): "deleted 1984 (70.1 GiB) · moved to Trash 0 · ran tools 3 · skipped 20 · failed 0", "Free space: 87.4 GiB → 164.7 GiB"; `df -h /` right after: 165 GiB available

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

**What changed.** `ddc clean` (`crates/ddc-cli/src/cmd/clean.rs`, `prompt.rs`): a fresh scan under the exclusive lock, a plan by group (`ddc-core/src/plan.rs`), interactive selection with a separate confirmation for `review`, `--yes` limited to items selected by default in the allowed classes, `--dry-run` without an executor. The executor (`ddc-core/src/exec/`) re-checks every item right before it acts (ADR "Cleanup safety model: risk classes, plan and confirmation" §5: identity of the finding and its parent, symlinks, protected folders, project roots, `.ddc-keep`, processes using it, a running owner app), then runs the tool's own command, renames to a tombstone and deletes, or moves to the Trash, and journals every action to `actions.jsonl`. After the first real run, owner apps are recognised by their app bundle (or the executable's name inside it) instead of a process-name prefix, and `ide.editor-cache` is split into one rule per editor.

**Why.** ADR "Cleanup safety model: risk classes, plan and confirmation": nothing is removed without a plan the user has seen, and a stale or swapped finding is skipped, not deleted. The owner change came from the live run: macOS's `CursorUIViewService` matched `Cursor` and held back 15 editor caches (2.7 GiB) while no editor ran.

**Tests.** `make check` on 2026-10-18: fmt, clippy `-D warnings`, 119 tests. This story's tests are in `crates/ddc-core/tests/clean_safety.rs`, `crates/ddc-cli/tests/clean_cli.rs` and `crates/ddc-core/src/sys/procs.rs`. `projectstore:reviewer` checked the owner amendment against its criteria; its should-fix (a renamed bundle was no longer recognised, which the old name check caught) was fixed and tested (`a_renamed_bundle_is_recognised_by_its_executable`).

**Live run (2026-10-18).** The owner's `ddc clean` on the reference machine: 2007 items, 100.7 GiB selected; deleted 1984 (70.1 GiB), 3 tool commands (`go clean -modcache`, `uv cache clean`, `go clean -cache`), 20 skipped, 0 failed; free space 87.4 → 164.7 GiB. The skips were 4 apps running (Spotify, Chrome, Brave, and the false `CursorUIViewService` match, since fixed) and 2 folders in use (`~/.npm/_npx` by `node`, a project's `node_modules` by `workerd`).

**Risks and follow-ups.**
- No output between the confirmation and the result: the backlog story "Progress output while ddc clean runs" (an idea from this run).
- Owner matching by `CFBundleIdentifier` would remove the rename and name-collision limits listed in Technical Notes; owners per known path would spare a new rule id per VS Code fork.
- Accepted delete-side limits: an app whose bundle and executable are both renamed, and `xcodebuild` between compiler steps, are not recognised as owners (only `safe` caches and DerivedData are affected).
- `local_snapshots` runs twice (before the prompt and inside `execute`).

## Technical Notes

- `std::fs::remove_dir_all` does not follow symlinks (since Rust 1.58.1, CVE-2022-21658), but the re-check is still needed.
- Trash: crate `trash` with `TrashContextExtMacos::set_delete_method(DeleteMethod::NsFileManager)` (no AppleScript and no Automation permission prompt; no "Put Back"). In tests — a double that moves items into `<tmp>/FakeTrash`: the real crate puts them into the real Trash regardless of `HOME`.
- Owners by bundle rather than process name: Windsurf's main binary is called `Electron`, Electron and Chrome helpers are `<App> Helper (Renderer)`, and macOS ships unrelated processes whose names start with an app's name (`CursorUIViewService`). The bundle is also what the user sees, so the skip reason names it. This replaces the name-only matching noted in the story "Detection rule catalog" (amended there); its `ide.editor-cache` row is now three rules. Journal lines written before the split keep the old id; nothing reads rule ids back from the journal or the saved scan.
- Accepted limits. On the skip side (a cache is kept): a different app whose bundle or executable name equals an owner's, or extends it after a space or dash, holds a cache back: `Cursor Pro.app` (a cursor highlighter), fleetdm's `Fleet Desktop.app` (for the JetBrains caches), `Google Chrome Canary.app`, `Beta.app` and `Google Chrome for Testing.app` (they keep their own caches), and `Visual Studio Code - Insiders.app` (it uses `Code - Insiders/`); the skip reason names the bundle, so such a case explains itself. `ide.jetbrains-cache` is one folder for every JetBrains IDE, so any of them still holds back all of it. On the delete side, accepted: an app whose bundle and executable were both renamed is not recognised, and Xcode's `xcodebuild` (under `Contents/Developer`) is not Xcode, as before this change; while it compiles, the in-use check catches the compiler by its arguments, but not between compiler steps. Both affect only `safe` caches and DerivedData.
- Follow-up: matching by `CFBundleIdentifier` (from the bundle's `Info.plist`, cached per bundle) would remove both the rename and the name-collision cases; owners per known path rather than per rule would spare a new rule id for every further VS Code fork.
- CLI tests run the binary with `env_clear()`, temporary `HOME`, `PATH` (empty or with fake `npm`/`go`), `TMPDIR` and `NO_COLOR=1`.

## Dependencies

- [[epics/DDC-001/stories/story-ddc-scan-report|ddc scan report]]

## Attachments

-

---

*Last updated: 2026-10-18*

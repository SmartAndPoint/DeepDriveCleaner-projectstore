---
type: story
id: "story-safe-cleanup-ddc-clean"
epic: "DDC-001"
title: "Safe cleanup: ddc clean"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-17
updated: 2026-10-17
external_refs: {}
tags: [cli, cleanup, safety]
code_refs: ["crates/ddc-core/src/plan.rs", "crates/ddc-core/src/exec/", "crates/ddc-core/src/sys/procs.rs", "crates/ddc-core/src/sys/fsops.rs", "crates/ddc-core/tests/clean_safety.rs", "crates/ddc-cli/src/cmd/clean.rs", "crates/ddc-cli/src/prompt.rs", "crates/ddc-cli/tests/clean_cli.rs"]
specs: []
started_at: "2026-10-17T10:16:47.568Z"
closed_at: null
plan_updated_at: "2026-10-17T10:16:47.568Z"
---

# Safe cleanup: ddc clean

| Field | Value |
|---|---|
| **Epic** | [DDC-001](../epic.md) |
| **Status** | in-progress |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

Removing the selected items under the rules of [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]: fresh scan → plan → selection → confirmation → execution with a re-check of every path and a journal.

## Decomposition

- [ ] `ddc clean` always rescans under an exclusive lock; a plan with selection by class, lock file and activity (`--min-age`, default 14 days); `user-data` does not enter the plan; the scan root does not enter the plan
- [ ] Filters: repeatable `--class` (without `--yes`, all classes except `user-data` are shown by default), `--category`, `--min-size`; `--dry-run` prints the plan and exits, no executor is created
- [ ] Interactive selection by group (multi-select); a separate confirmation for `review`; a warning that the Trash does not free space
- [ ] `--yes`: only items selected by default from the classes in `--class` (default `safe`); `review` in `--class` together with `--yes` is an error; without a TTY, `--yes` or `--dry-run` is required
- [ ] Executor per method: `native` (absolute path, no shell, stdin closed, output to a journal file, timeout) → `delete` (rename to a tombstone, removal without following symlinks, on `EACCES` — `u+w` and retry) → `trash` (`NsFileManager`; for all `review`, including `rebuildable` downgraded to `review`; `--permanent` replaces it with `delete`) → `native-only` (only the command while the daemon is running; a command failure does not fall through to deletion)
- [ ] Re-check: exists, is a directory, not a symlink, `(dev, ino)` of the finding and its parent match, `Catalog::verify` passes, inside the scan root and not equal to it, not a protected folder (by path and by `(dev, ino)`), not a project root, no `.git` at the top level, no `.ddc-keep` in the finding or its ancestors; tombstone via `renameat` from the verified parent descriptor
- [ ] `in-use` check: the path from `KERN_PROCARGS2` (before symlink resolution), arguments, `VIRTUAL_ENV`/`CONDA_PREFIX`, the working directory (`PROC_PIDVNODEPATHINFO`) of any user process inside the finding; relative paths are joined with the process's working directory; owner-process check
- [ ] All side effects go through traits (`CommandRunner`, `Trasher`, `ProcessProbe`, `Journal`) with fakes in tests; the executor refuses to run as root
- [ ] Warning about local Time Machine snapshots
- [ ] Journal `<log-dir>/actions.jsonl` with `schema_version`; summary: deleted / in Trash / freed per `statfs`

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

Files touched: `crates/ddc-core/src/{plan.rs,exec/,sys/procs.rs,sys/fsops.rs,scan/,model.rs,report/mod.rs}`, `crates/ddc-core/tests/clean_safety.rs`, `crates/ddc-cli/src/{cmd/clean.rs,prompt.rs,cli.rs}`, `crates/ddc-cli/tests/clean_cli.rs`, `Cargo.toml` (`trash`).

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test: a path replaced by a symlink between scan and cleanup is not deleted; the reason is in the journal
- [ ] Test: a directory with the same path and marker but a different inode is not deleted (`changed-since-scan`)
- [ ] Test at the executor API level: `home`, the scan root, `home/Documents` and its variant with different letter case, and a project root are rejected even if passed into the plan directly
- [ ] Test: `--yes` deletes only selected `safe`; with `--class safe --class rebuildable` — also selected `rebuildable`; `--yes --class review` is an error; `review` is not deleted without interaction
- [ ] Test: the executor refuses to apply `delete`/`trash` to a `native-only` rule and any action to `user-data`, even if they are slipped into the plan (`action-not-allowed`); a failed native command does not lead to deletion
- [ ] Test: a venv without a lock file goes to the Trash, and with `--permanent` it is deleted (executor, not only the plan); a directory swapped during the tool probe is not deleted; a partially unreadable finding is not deleted
- [ ] Test: `--dry-run` does not change the tree (paths, types, inodes match before/after) and does not write the journal
- [ ] Test: a tree with `0555` directories is deleted; an interrupted deletion leaves a tombstone that the next scan finds as `safe`
- [ ] Test: a process launched via `<tmp>/.venv/bin/sleep` (a symlink to `/bin/sleep`) blocks deletion (`in-use`); the same for launching `./.venv/bin/sleep` with working directory `<tmp>` and for a working directory inside the finding
- [ ] Test: a `.ddc-keep` created in a finding after the scan cancels its deletion
- [ ] Test: each action produces one journal line with path, rule, class, method, sizes, result
- [ ] Test: `clean` does not start while the exclusive lock is held
- [ ] Manual check on the reference machine: cleaning `safe` frees space, `df` confirms

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- `std::fs::remove_dir_all` does not follow symlinks (since Rust 1.58.1, CVE-2022-21658), but the re-check is still needed.
- Trash: crate `trash` with `TrashContextExtMacos::set_delete_method(DeleteMethod::NsFileManager)` (no AppleScript and no Automation permission prompt; no "Put Back"). In tests — a double that moves items into `<tmp>/FakeTrash`: the real crate puts them into the real Trash regardless of `HOME`.
- CLI tests run the binary with `env_clear()`, temporary `HOME`, `PATH` (empty or with fake `npm`/`go`), `TMPDIR` and `NO_COLOR=1`.

## Dependencies

- [[epics/DDC-001/stories/story-ddc-scan-report|ddc scan report]]

## Attachments

-

---

*Last updated: 2026-10-17*

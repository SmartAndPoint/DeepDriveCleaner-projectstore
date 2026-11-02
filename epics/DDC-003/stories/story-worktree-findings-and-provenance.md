---
type: story
id: "story-worktree-findings-and-provenance"
epic: "DDC-003"
title: "Worktree findings and provenance"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-11-01
external_refs: {}
tags: [rules, worktrees, safety, git]
code_refs: []
specs: []
started_at: "2026-11-01T15:56:14.955Z"
closed_at: null
plan_updated_at: "2026-11-01T15:56:14.955Z"
---

# Worktree findings and provenance

| Field | Value |
|---|---|
| **Epic** | [DDC-003](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

A linked git worktree becomes a finding. This story is §1 and §2 of the ADR "Agent footprint: worktrees, harness data and bounded consent for agents": the `git.worktree` and `git.worktree-orphan` rules, the `origin` provenance and the `worktree` state field, the `worktree_of` grouping, the git probe contract, the six conditions under which a harness's worktree is `rebuildable` (ignored content and submodules included), the `contained` plan rule, and removal through `git worktree remove` alone. After it, `ddc scan` shows an agent's dead worktrees under their project and `ddc clean --yes --class rebuildable` removes them through git. Nothing agent-facing is added yet; `footprint` and `--agent` are separate stories.

The rule fires for every linked worktree in the scan roots. Only a harness's worktree can be `rebuildable`; a person's own stays `review`.

## Decomposition

Rough until the plan gate.

- [ ] Model: `Finding.origin: Option<Origin { harness, kind }>`, `Finding.worktree: Option<WorktreeState { branch, detached, clean, ignored_content, own_commits, submodules, locked, in_use, idle_days, probe }>` and `Project.worktree_of: Option<PathBuf>`, all optional; the `Note` enum is **not** touched, so `schema_version` does not rise for them; `Origin` itself arrives with the story "Harness data rules" (ADR §3, settled at the plan gate 2026-10-30), and this story fills it for worktrees; `docs/scan.schema.json` regenerated, plus a fixture report written by the current release (schema 4) that the new model still decodes
- [ ] Matcher `Worktree`: `.git` is a file with `gitdir:`; the gitdir exists → `git.worktree`, else `git.worktree-orphan`; the main checkout from `<gitdir>/commondir`. Checked in `after_read` before the project-manifest exclusion, because a worktree is a project root and must still be claimed as a **region**: rules keep firing inside it, findings inside get `project.root = worktree` and `project.worktree_of = main`
- [ ] Provenance: `origin.harness = claude-code` when the path is under `<main>/.claude/worktrees/` or the gitdir carries the marker Claude Code writes (decided at the plan gate by reading a real worktree's gitdir on the reference machine); otherwise `git`
- [ ] Git probe module (`ddc-core/src/git.rs`, no `unsafe`): one function that runs `git` through the `CommandRunner` with `--no-optional-locks`, `GIT_OPTIONAL_LOCKS=0`, `GIT_TERMINAL_PROMPT=0` and the `-c` overrides of the ADR (`core.fsmonitor=false`, `core.untrackedCache=false`, `core.hooksPath=/dev/null`, `status.submoduleSummary=false`, `core.pager=cat`, `core.askPass=`), a 30 s timeout, and a typed result; any failure is `Probe::Unknown`, never retried with fewer guards. Used by scan, `footprint` (next story) and the executor's re-check
- [ ] Classification (`rules/classify.rs`), for `git.worktree` with a known harness only, in the ADR's order, stopping at the first failure and recording it in `worktree`: unlocked and not in use → idle for `--worktree-min-age` (default 2 days) → no initialized submodule → clean including ignored content → no own commits. Probes run in parallel across candidates and only for candidates
  - **Ignored content**: `status --porcelain=v1 --untracked-files=all --ignored=matching`; each `!!` path must lie inside a claimed inner finding of class `safe` or `rebuildable` with method `delete`, **or** be byte-identical to the file at the same relative path in the main checkout (`.worktreeinclude` copies). Anything else → `review`, `ignored_content` lists up to 20 paths and a count
  - **Submodules**: `.gitmodules` present and any listed path has a `.git` → `review`, `submodules: true`
  - **Own commits**: survivors = `refs/heads/*` minus the candidate branch, `refs/remotes/*`, `refs/tags/*`, and every other worktree's `HEAD` commit from `worktree list --porcelain`; `rev-list --count <candidate HEAD commit> --not <survivors>` must be 0. No `--all`, no `--branches`. Works the same for a detached `HEAD` and from any checkout
- [ ] Plan (`plan.rs`): an item inside a selected `git.worktree` item is dropped with the new reason `contained`; the worktree's reclaimable bytes include them
- [ ] Removal: `Removal::Native` for `git.worktree` with `git -C <main> worktree remove <path>` under the probe contract and **nothing else**: no `prune`, no `--force`. Location probe: `git -C <main> worktree list --porcelain` **lists** the path (a new `LocationCompare::Lists`, since the probe prints several lines), fallback `Skip` (report only; there is no `Fallback::None`). Settled at the plan gate (the `LocationCompare::Lists` idea above is dropped): a `Location::GitWorktree` variant handled by its own function in `exec/native.rs`, as `Location::Conda` is, delegating to `git.rs`, instead of `{main}` substitutions and `LocationCompare::Lists`. `git.worktree-orphan`: `trash`, `--permanent` → `delete`
- [ ] Executor re-check (`exec/guard.rs`, `Catalog::verify`): all six conditions are re-run before the native command through the same probe module; a `git.worktree` item whose plan action is `delete` or `trash` is `action-not-allowed`; the "no `.git` at the top level" refusal stays for every other rule
- [ ] Hints: `git.worktree` → "removed with git worktree remove; branch <branch> is kept, git branch -d to drop it"; a `review` worktree's hint names its first failed condition and, for `ignored_content`, the first path
- [ ] Render: a worktree's artifacts are shown indented under the worktree, and the worktree under its main project; a `review` worktree shows its reason
- [ ] `--worktree-min-age <days>` on `scan` and `clean`; `settings.json` key `worktree_min_age_days` once DDC-002's settings story lands (not a dependency: the flag is enough here)
- [ ] README: the worktree rule, the six conditions in one paragraph, the two-day window, `.ddc-keep` and `--worktree-min-age`

## Implementation Plan

Planned 2026-11-01 with the planner against `origin/main` 1cca33e. Four PRs, each with its own version bump and release notes. PR 1 alone answers the tester who saw only the `.venv` inside agent worktrees.

**Plan-gate decisions** (recorded in the ADR as a dated amendment of §2):
1. Provenance markers as in the Technical Notes.
2. Fallback `Skip` with `tool-not-ready` (no new skip reason, no schema bump of the clean, journal or progress documents).
3. The guard's `.git` refusals get a carve-out scoped to `git.worktree` + `Native` and `git.worktree-orphan` + `trash`/`delete`.
4. A copied or moved worktree (`cp -r`, `mv`) points to an existing gitdir whose back-pointer names another place. `git.worktree` requires the back-pointer `<gitdir>/gitdir`, resolved against the gitdir when relative (`worktree.useRelativePaths`), to name the same file as `<path>/.git` by identity (`dev`, `ino`), not by string, so case-folding on APFS and relative links do not mismatch. A mismatch is `review` with `WorktreeState.linked: false` and the hint "moved or copied: `git worktree repair` relinks it"; it is never removed through git. The `worktree list --porcelain` comparison of PR 3 uses the same identity.
5. `classify::downgraded` (`app_owned` for hidden top-level folders: `~/.codex/worktrees`, `~/.cursor/worktrees`) is skipped for the two worktree rules **and for the findings inside a worktree region**: the worktree's class comes from the six conditions alone, and its inner artifacts get the class their own rules give, as in any project.
6. Listed consent (`clean --items`, the app) on a `git.worktree`: conditions 1 (known harness) and 3 (idle) are waived, a person decides; conditions 2, 4, 5 and 6 are checked for a listed worktree in `plan_listed`'s pruned walk (so the dry run and the confirmation sheet already show a refusal) and again by the executor, so `git worktree remove` never deletes unique ignored content or the only HEAD-reachable copy of a commit, whoever asked. (Commits only in the worktree's own reflog go with it; that is inherent to `git worktree remove`.)
8. **Skip reasons.** A listed worktree that fails a condition is skipped with a **new** reason, `worktree-kept { condition, detail }`; the clean report, the journal and the progress events bump their schema version for it, per the Rust core ADR's policy, in PR 3. An item inside a worktree that a clean removed through git is skipped `inside-another-item`, as inside a trashed folder: the executor's tracking of removed parents extends to a `Native` outcome of `git.worktree`. `--items` naming both a worktree and an item inside it plans the worktree and refuses the inner item `inside-another-item`, as for folders. `changed-since-scan` stays for an item whose identity changed.
9. **Activity.** For the two worktree rules `report::finding` skips the `--min-age`/`ActiveProject` gate, which reads the main checkout's activity: the worktree's `Project.last_activity` and `idle_days` are its own (`last_activity(<worktree>)`, following `gitdir:`), and condition 3 against `--worktree-min-age` is its only idle test. A dead agent worktree of a project committed to yesterday is still eligible.
10. **Which git.** `git.rs` resolves its own executable and never runs `/usr/bin/git`, the `xcrun` shim that offers to install the developer tools (a LaunchAgent's `PATH` starts with `/usr/bin`, so `SystemCommands::find` would pick it): candidates are `Env::tool_dirs()` without `/usr/bin`; else `xcode-select -p` (a real binary; it exits 2 without a dialog when unset) and `<developer dir>/usr/bin/git` run directly; else `probe: unknown`. The scan's probes, `Location::GitWorktree` and the removal all go through it.
7. The ADR is `proposed`; §1 and §2 are accepted at this gate, as §3 was at its own.

*Progress, 2026-11-02: PR 1 shipped in 0.15.2 (code PRs #69, #70). Lesson from it: a test fixture that ran git from the pre-push hook inherited `GIT_DIR` and set `core.bare=true` in the main checkout; the fixture and, in PR 2, every production probe clear inherited `GIT_*` variables.*

**PR 1. Recognition, read-only.**
- `model.rs`: `WorktreeState { …, probe: known|unknown }`, `Finding.worktree`, `Project.worktree_of`, all optional with `skip_serializing_if`; `SCHEMA_VERSION` stays 4.
- `rules/mod.rs`: `Matcher::Worktree`, `Matcher::WorktreeOrphan`. New `rules/worktree.rs`: `recognize(path) -> Option<Linked { gitdir, main, harness }>`, reusing `report::project::git_dir`, reading `commondir` (a submodule's gitdir has none, so it never matches) and the back-pointer; `main` from `commondir`, never a literal `.git/worktrees/` string (a bare main repo).
- `rules/visitor.rs::matched`: the worktree matcher before `claimable()`, returning a new `scan::Post::Region(Tag)` handled in `scan/walker.rs` beside `Pre::Region`; the `.ddc-keep` check stays after it (a kept worktree becomes `user.kept`, its inner findings with it).
- `catalog/shared/projects.rs`: the two rules on macOS, Linux and Windows (`COUNT`); `rules/listing.rs` GROUPS.
- `report/mod.rs::finding`: per-finding `origin` (today copied from the rule); the worktree's `project.root` = main; inner findings get `worktree_of`.
- `Removal::None` until PR 3, so nothing can list one for removal; the finding is `not_selected: review` with the hint "a linked worktree of `<main>`; ddc removes worktrees through `git worktree remove` in a later release", so the row has a reason. A `git`-origin worktree's hint reads as the person's own ("yours; list it to remove it" from PR 3 on), never as a failed condition; an orphan's hint names both causes ("its repository is gone, or on a volume that is not mounted or was moved: `git worktree repair`").
- `docs/scan.schema.json` (`DDC_UPDATE_SCHEMA=1`), `bench/golden/rules.json` (`make golden-record`, diff from the new rules only).

**PR 2. Probes and classification.**
- New `crates/ddc-core/src/git.rs`, shaped like `conda.rs` behind `CommandRunner`: `PROBE_CONFIG` constant with a test per entry, an injectable `PROBE_TIMEOUT` (moved out of `native.rs`), typed results, `Probe::Unknown` on any failure.
- macOS: `/usr/bin/git` is an `xcrun` shim that opens the "install developer tools" dialog without the Command Line Tools; probes run only when a real git is present (the CLT or Xcode path exists, or a non-shim `git` on `PATH`), else `probe: unknown`. A background scan never opens a dialog.
- `ScanOptions`: `commands` (an `Arc<dyn CommandRunner + Send + Sync>`, default `SystemCommands::new(env)`) and `worktree_min_age`. A Mutex-based fake for the parallel probes (`FakeCommands` is `RefCell`).
- The six conditions in `rules/worktree.rs`, called from `classify.rs`; in-use through `exec::guard::user_of` with `ScanOptions.processes`; idle through `project::last_activity`; ignored-content accounting against the claimed inner owners plus a byte compare with `<main>/<rel>`; probes in parallel with `rayon`, only for candidates.
- `report::finding`: decision 9 (no `ActiveProject` gate for worktrees, own activity); a failed condition → `NotSelected::Review` (no new variant); hints. CLI `--worktree-min-age` on `scan` and `clean`.

**PR 3. Removal.**
- `class.rs`: `Location::GitWorktree`; `native.rs`: `git -C <main> worktree list --porcelain` and `git -C <main> worktree remove <path>` through `git.rs`, `<main>` re-derived from disk and compared with the report's `worktree_of`.
- `Catalog::verify` and `guard.rs::check`: the scoped carve-out and the condition re-check (decision 6 for Listed consent), skipping `worktree-kept` (decision 8, the schema bumps); the executor's removed-parent tracking covers `git.worktree`; `plan_listed` runs the listed worktree's conditions; `exec/mod.rs::action_allowed`: `Native` only for `git.worktree`.
- `plan.rs`: `contained` items dropped, their bytes added to the worktree's (a region finding's own bytes exclude its inner claims).
- Orphan: `trash`, `delete` under `--permanent`.

**PR 4.** `render.rs::report`: the worktree under its project, inner findings indented; README section (`readme_cli.rs` requires every option); the reference-machine check.

**Pitfalls carried into the PRs.** Real-git fixtures run with `GIT_CONFIG_GLOBAL=/dev/null`, `GIT_CONFIG_NOSYSTEM=1`, `-c user.name/email`, `init -b main`. Windows: the scanner is unported, so tests calling `common::unsupported` go into `.config/native-skips/windows.txt`; Linux stays green (git and `/proc` work), a test goes into `linux.txt` only if truly macOS-only; `git.rs` compiles on Windows. Coverage floors: `plan.rs` 93.9, `classify.rs` 95.6, `guard.rs` 90.6, `exec/mod.rs` 89.3, `ddc-core` 92.9.

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test (a real `git` repository in a temp tree, `TreeBuilder` plus `git init`/`git worktree add`): a worktree under `<main>/.claude/worktrees/x` is a `git.worktree` finding with `origin.harness = claude-code`; one made with `git worktree add ../y` is `git.worktree` with `origin.harness = git`; a copied directory whose `.git` file points to a missing gitdir is `git.worktree-orphan`; the main checkout itself is not a finding
- [ ] Test: a `.venv` with `pyvenv.cfg` and `uv.lock` inside the harness worktree is a `python.venv` finding with `project.root` = the worktree and `project.worktree_of` = the main checkout; `report::project::last_activity` for it reads the worktree's gitdir, not the main checkout's
- [ ] Test (probe contract): with `core.fsmonitor` set in the repository config to a script that writes a marker file, a full scan and a re-check run and the marker is never written; every probe invocation recorded by the fake `CommandRunner` carries `--no-optional-locks` and each `-c` override of the constant; after two scans of an untouched worktree the index bytes and mtime are unchanged and both reports have the same `worktree` state and `selected`
- [ ] Test (probe failure): a `git` on `PATH` that hangs, one that exits 128, and one that prints garbage each leave the worktree `review` with `probe: unknown`, within the timeout, and the executor skips the item
- [ ] Test, one case per condition, each leaving every other condition satisfied: a harness worktree that is unlocked, not in use, three days idle, without submodules, clean with no ignored content and without own commits is `rebuildable` and `selected`; `<gitdir>/locked` present → `locked: true`; a process whose cwd is inside it (the fake `ProcessTable`) → `in_use`; touched one hour ago → idle below the window; an initialized **clean** submodule → `submodules: true` and not selected, and `git worktree remove` is never attempted; a submodule with changes → `clean: false`; one untracked file → `clean: false`; `origin.harness = git` → `review` with no probe run at all (the fake `CommandRunner` records no call)
- [ ] Test (ignored content): a worktree whose only ignored paths are a `.venv` with `pyvenv.cfg` and `uv.lock` (a `rebuildable`/`delete` finding) and a `.env` byte-identical to `<main>/.env` is eligible; the same worktree with (a) a unique ignored `.env`, (b) an ignored `state/local-only.txt`, (c) an ignored `.venv` **without** a lock file (its own rule says `review`) is `review` with `ignored_content` naming the path, the `.venv` of case (c) is still its own `review` finding, and after `ddc clean --yes --class rebuildable` all three files still exist
- [ ] Test (own commits): named branch merged into `main` with its remote deleted → `own_commits: 0`, eligible; named branch with one commit not on any surviving ref → `own_commits: 1`, `review`; detached `HEAD` on a commit reachable from `main` → eligible; detached `HEAD` on a unique commit → `own_commits: 1`, `review`; each of the four gives the same answer when the probe is run from the main checkout and from another linked worktree; a commit that is only on **another worktree's** detached `HEAD` counts as surviving
- [ ] Test (`plan.rs`): with the worktree selected, its inner `.venv` item is dropped as `contained` and the plan's total counts the worktree's bytes once; with the worktree `review`, the inner `.venv` is planned on its own as today
- [ ] Test (`clean_safety.rs`): removing a selected worktree runs exactly one git command through the `CommandRunner`, `git -C <main> worktree remove <path>` under the probe contract, never `prune`, never `--force`; afterwards `git worktree list` does not list it, `<main>/.git/worktrees/<name>` is gone, the branch `worktree-<name>` still exists, and the journal line carries the command's tail. A second worktree whose directory was moved away before the clean keeps its `<main>/.git/worktrees/<other>` entry, `HEAD`, index and reflog intact. With the location probe not listing the path, the item is skipped `tool-not-ready` (program `git`, detail "no git lists this folder") and nothing runs
- [ ] Test: a `git.worktree` item planned with action `delete` or `trash` is skipped `action-not-allowed`; a directory with `.git` at its top level that is **not** a linked worktree is still refused as before (the existing test in `clean_safety.rs` keeps passing)
- [ ] Test: the executor's re-check runs all six conditions again through the probe module; a worktree made dirty, given a unique ignored file, or given a commit between scan and clean is skipped `changed-since-scan`
- [ ] Test (schema): `--json` validates against the regenerated `docs/scan.schema.json`, the committed schema equals the generated one, `Note` has no new variant (a test enumerates the variants against a fixed list), and a fixture report written by the current release still decodes with the new model and a report with `origin`/`worktree` still decodes with a strict reader of the fixture's model after unknown fields are dropped
- [ ] Test (`scan_cli.rs`, `clean_cli.rs`): `--worktree-min-age 0` selects a clean harness worktree touched a minute ago; the default leaves it below the window
- [ ] On the reference machine: `ddc scan ~/Projects` lists every linked worktree `git worktree list` reports for the projects there, and the selected ones are exactly those a manual check (`git status --ignored`, the surviving-ref `rev-list`, `ls .git/worktrees/*/locked`, `.gitmodules`) agrees are dead; result in the final summary with counts, bytes and the ignored-content reasons that held worktrees back
- [ ] Test (decision 4): a worktree whose back-pointer is relative (`git worktree add --relative-paths`) is `git.worktree` and linked; one reached through a path differing only in case is linked; a `cp -r` copy and an `mv`-moved worktree are `review` with `linked: false` and the repair hint, and no git command is planned for them
- [ ] Test (decision 6, `Consent::Listed`): a person's own (`origin.harness = git`) clean worktree, listed, is removed through git; the same worktree with a unique ignored file, listed, is skipped `worktree-kept` naming condition 5, in the dry run and in the clean; a harness worktree touched an hour ago, listed, is removed
- [ ] Test (decision 8): `--items` with a worktree and its `.venv` removes the worktree and skips the `.venv` `inside-another-item`; `docs/clean.schema.json`, the journal and the progress schema carry `worktree-kept` with their bumped versions, and DDCKit decodes them
- [ ] Test (decision 9): a clean harness worktree idle three days whose main checkout was committed to a minute ago is `rebuildable` and selected, its `last_activity` the worktree's own
- [ ] Test (decision 10): with a fake runner whose only `git` is `/usr/bin/git` and no developer directory, a scan records no git call and the worktree is `probe: unknown`; with a developer directory, the developer directory's git is the one called
- [ ] `make check` passes

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- `report::project::git_dir` already follows `gitdir:` for activity; the matcher reuses it and adds `commondir`.
- The catalog rule "a directory with `.git` at its top level is never claimed" (story "Detection rule catalog") needs one carve-out: a **file** `.git` with `gitdir:` into a gitdir that has a `commondir` file is the worktree rule's marker (a bare main repository's worktree gitdir is `<bare>/worktrees/<n>`, so the discriminator is `commondir`, not a `.git/worktrees/` string). A directory `.git`, or a gitdir without `commondir` (a submodule's `.git/modules/<n>`), keeps today's behaviour: not claimed. A submodule checkout is therefore never a `git.worktree`.
- *Corrected at the plan gate, 2026-11-01:* the executor's "not a project root" refusal (`exec/guard.rs::check`) applies to **every** action whenever `<path>/.git` exists, `Native` included, and before it `Catalog::verify` → `claimable()` refuses any directory with `.git` (`rule-mismatch`). Both get a carve-out scoped to `git.worktree` with `Action::Native`, and to `git.worktree-orphan` with `trash`/`delete`; every other rule keeps both refusals, and the test for `delete`/`trash` on a `git.worktree` asserts `action-not-allowed`.
- `NativeCommand` substitutes `{name}` and `{path}` but not `{main}`, and the location probe runs with static args, no environment and one parsed path, so it cannot carry the probe contract's environment. Settled at the plan gate: `Location::GitWorktree`, handled in `exec/native.rs` through `git.rs` (see the plan).
- **Why the probe contract exists** (PR #1 review): a plain `git status` runs the `core.fsmonitor` helper the repository configures and, through its background refresh, rewrites the index whose mtime is the activity signal; both were reproduced on git 2.54. `--no-optional-locks` stops the refresh; the `-c` overrides stop the helpers; the command line outranks every config file. `core.hooksPath=/dev/null` is belt and braces: no probe runs a hook-bearing command, but `worktree remove` is not a probe and the override costs nothing.
- **Why `--ignored=matching`** (PR #1 review): `status` lists no ignored path by default, and `git worktree remove` without `--force` deleted an ignored `.env` and `state/local-only.txt` from a clean worktree in the reviewer's fixture. `matching` lists each ignored path rather than collapsing ignored directories, which the per-path accounting needs; on a worktree with a large ignored tree (`node_modules`) the listing is long, and the accounting short-circuits once a path is inside a claimed `delete` finding.
- **Why an explicit surviving-ref set** (PR #1 review): `rev-list --all` includes `HEAD`, and traversal includes other worktrees' `HEAD`s, so both `--all` sketches of the first draft counted a unique commit as preserved. Naming the survivors and the candidate's commit by hash makes the answer independent of where the probe runs and covers a detached `HEAD`, whose commit nothing else holds once the worktree is gone.
- **Why no `prune`** (PR #1 review): `git worktree prune --dry-run --verbose` selected another, moved worktree's administrative directory in the reviewer's fixture; `remove` already deletes its own `.git/worktrees/<name>`.
- **Why `submodules` is a condition** (PR #1 review): `git worktree remove` exits 128 with "working trees containing submodules cannot be moved or removed" for a clean worktree with an initialized submodule, and `--force` is forbidden; reporting it as reclaimable would promise what the executor cannot do.
- `git status` on a large tree can take seconds; the probes run only for candidates that are unlocked, not in use and past the idle window, and in parallel. The 30 s timeout is the existing probe timeout in `exec/native.rs`.
- **Provenance markers, read on the reference machine at the plan gate (2026-11-01).** In `<main>/.git/worktrees/<name>/`: Claude Code's CLI writes `CLAUDE_BASE` (the 40-hex base commit, no newline) and locks a live session's worktree with `locked` = `claude session <name> (pid N start <date>)`; Claude Desktop writes an empty `claude-desktop-worktree`; Codex writes `codex-thread.json` (`ownerThreadId`, `version`) and a `config.worktree` with a `[codex]` section, its worktrees under `~/.codex/worktrees/<id>/<repo>`; Cursor writes none, its worktrees under `~/.cursor/worktrees/<repo>/<id>`. Older or agent-made worktrees under `.claude/worktrees/` carry no marker, so the path stays a signal. Rule: `claude-code` when the gitdir has a valid `CLAUDE_BASE` or `claude-desktop-worktree`, or the path is under `<main>/.claude/worktrees/`; `codex` from `codex-thread.json` (shown, but `review` in this story); otherwise `git`.
- An administrative entry whose worktree directory is gone (`<main>/.git/worktrees/<x>/gitdir` pointing to a missing path) is not a directory and not a finding; it is what the "no `prune`" rule protects.

## Dependencies

- [[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]]
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]
- [[epics/DDC-001/stories/story-detection-rule-catalog|Detection rule catalog]]
- [[epics/DDC-001/stories/story-safe-cleanup-ddc-clean|Safe cleanup: ddc clean]]

## Attachments

- Claude Code worktrees: https://code.claude.com/docs/en/worktrees

---

*Last updated: 2026-11-01*

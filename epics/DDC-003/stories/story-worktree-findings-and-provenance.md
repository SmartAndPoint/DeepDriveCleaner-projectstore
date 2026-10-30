---
type: story
id: "story-worktree-findings-and-provenance"
epic: "DDC-003"
title: "Worktree findings and provenance"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-30
external_refs: {}
tags: [rules, worktrees, safety, git]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
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

- [ ] Model: `Finding.origin: Option<Origin { harness, kind }>`, `Finding.worktree: Option<WorktreeState { branch, detached, clean, ignored_content, own_commits, submodules, locked, in_use, idle_days, probe }>` and `Project.worktree_of: Option<PathBuf>`, all optional; the `Note` enum is **not** touched, so `schema_version` does not rise for them; `Origin` itself arrives with the story "Harness data rules" (ADR §3, settled at the plan gate 2026-10-30), and this story fills it for worktrees; `docs/scan.schema.json` regenerated, plus a fixture report of the committed version-1 schema that a strict reader of today's model still decodes
- [ ] Matcher `Worktree`: `.git` is a file with `gitdir:`; the gitdir exists → `git.worktree`, else `git.worktree-orphan`; the main checkout from `<gitdir>/commondir`. Checked in `after_read` before the project-manifest exclusion, because a worktree is a project root and must still be claimed as a **region**: rules keep firing inside it, findings inside get `project.root = worktree` and `project.worktree_of = main`
- [ ] Provenance: `origin.harness = claude-code` when the path is under `<main>/.claude/worktrees/` or the gitdir carries the marker Claude Code writes (decided at the plan gate by reading a real worktree's gitdir on the reference machine); otherwise `git`
- [ ] Git probe module (`ddc-core/src/git.rs`, no `unsafe`): one function that runs `git` through the `CommandRunner` with `--no-optional-locks`, `GIT_OPTIONAL_LOCKS=0`, `GIT_TERMINAL_PROMPT=0` and the `-c` overrides of the ADR (`core.fsmonitor=false`, `core.untrackedCache=false`, `core.hooksPath=/dev/null`, `status.submoduleSummary=false`, `core.pager=cat`, `core.askPass=`), a 30 s timeout, and a typed result; any failure is `Probe::Unknown`, never retried with fewer guards. Used by scan, `footprint` (next story) and the executor's re-check
- [ ] Classification (`rules/classify.rs`), for `git.worktree` with a known harness only, in the ADR's order, stopping at the first failure and recording it in `worktree`: unlocked and not in use → idle for `--worktree-min-age` (default 2 days) → no initialized submodule → clean including ignored content → no own commits. Probes run in parallel across candidates and only for candidates
  - **Ignored content**: `status --porcelain=v1 --untracked-files=all --ignored=matching`; each `!!` path must lie inside a claimed inner finding of class `safe` or `rebuildable` with method `delete`, **or** be byte-identical to the file at the same relative path in the main checkout (`.worktreeinclude` copies). Anything else → `review`, `ignored_content` lists up to 20 paths and a count
  - **Submodules**: `.gitmodules` present and any listed path has a `.git` → `review`, `submodules: true`
  - **Own commits**: survivors = `refs/heads/*` minus the candidate branch, `refs/remotes/*`, `refs/tags/*`, and every other worktree's `HEAD` commit from `worktree list --porcelain`; `rev-list --count <candidate HEAD commit> --not <survivors>` must be 0. No `--all`, no `--branches`. Works the same for a detached `HEAD` and from any checkout
- [ ] Plan (`plan.rs`): an item inside a selected `git.worktree` item is dropped with the new reason `contained`; the worktree's reclaimable bytes include them
- [ ] Removal: `Removal::Native` for `git.worktree` with `git -C <main> worktree remove <path>` under the probe contract and **nothing else**: no `prune`, no `--force`. Location probe: `git -C <main> worktree list --porcelain` **lists** the path (a new `LocationCompare::Lists`, since the probe prints several lines), fallback `None` (report only). `NativeCommand` needs `{main}` and `{path}` substitutions or a small variant. `git.worktree-orphan`: `trash`, `--permanent` → `delete`
- [ ] Executor re-check (`exec/guard.rs`, `Catalog::verify`): all six conditions are re-run before the native command through the same probe module; a `git.worktree` item whose plan action is `delete` or `trash` is `action-not-allowed`; the "no `.git` at the top level" refusal stays for every other rule
- [ ] Hints: `git.worktree` → "removed with git worktree remove; branch <branch> is kept, git branch -d to drop it"; a `review` worktree's hint names its first failed condition and, for `ignored_content`, the first path
- [ ] Render: a worktree's artifacts are shown indented under the worktree, and the worktree under its main project; a `review` worktree shows its reason
- [ ] `--worktree-min-age <days>` on `scan` and `clean`; `settings.json` key `worktree_min_age_days` once DDC-002's settings story lands (not a dependency: the flag is enough here)
- [ ] README: the worktree rule, the six conditions in one paragraph, the two-day window, `.ddc-keep` and `--worktree-min-age`

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

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
- [ ] Test (`clean_safety.rs`): removing a selected worktree runs exactly one git command through the `CommandRunner`, `git -C <main> worktree remove <path>` under the probe contract, never `prune`, never `--force`; afterwards `git worktree list` does not list it, `<main>/.git/worktrees/<name>` is gone, the branch `worktree-<name>` still exists, and the journal line carries the command's tail. A second worktree whose directory was moved away before the clean keeps its `<main>/.git/worktrees/<other>` entry, `HEAD`, index and reflog intact. With the location probe not listing the path, the item is skipped with the location mismatch and nothing runs
- [ ] Test: a `git.worktree` item planned with action `delete` or `trash` is skipped `action-not-allowed`; a directory with `.git` at its top level that is **not** a linked worktree is still refused as before (the existing test in `clean_safety.rs` keeps passing)
- [ ] Test: the executor's re-check runs all six conditions again through the probe module; a worktree made dirty, given a unique ignored file, or given a commit between scan and clean is skipped `changed-since-scan`
- [ ] Test (schema): `--json` validates against the regenerated `docs/scan.schema.json`, the committed schema equals the generated one, `Note` has no new variant (a test enumerates the variants against a fixed list), and a fixture report written by the current release still decodes with the new model and a report with `origin`/`worktree` still decodes with a strict reader of the fixture's model after unknown fields are dropped
- [ ] Test (`scan_cli.rs`, `clean_cli.rs`): `--worktree-min-age 0` selects a clean harness worktree touched a minute ago; the default leaves it below the window
- [ ] On the reference machine: `ddc scan ~/Projects` lists every linked worktree `git worktree list` reports for the projects there, and the selected ones are exactly those a manual check (`git status --ignored`, the surviving-ref `rev-list`, `ls .git/worktrees/*/locked`, `.gitmodules`) agrees are dead; result in the final summary with counts, bytes and the ignored-content reasons that held worktrees back
- [ ] `make check` passes

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- `report::project::git_dir` already follows `gitdir:` for activity; the matcher reuses it and adds `commondir`.
- The catalog rule "a directory with `.git` at its top level is never claimed" (story "Detection rule catalog") needs one carve-out: a **file** `.git` with `gitdir:` into `<x>/.git/worktrees/` is the worktree rule's marker. A directory `.git`, or a file `.git` pointing anywhere else (a submodule's `gitdir: ../.git/modules/…`), keeps today's behaviour: not claimed. A submodule checkout is therefore never a `git.worktree`.
- The executor's "not a project root" and "no `.git` at the top level" refusals (safety ADR §5) apply to `delete`/`trash`; `git.worktree` uses `Native` with `fallback` report-only, so the refusals are never reached for it, and the test for `delete`/`trash` on a worktree asserts the executor's `action_allowed` check instead.
- `NativeCommand` today substitutes only `{name}`; the worktree command needs the main checkout and the full path. Either add `{main}` and `{path}` substitutions or a `NativeCommand::worktree` variant; the plan gate decides. `LocationProbe` compares one path from stdout; `git worktree list --porcelain` prints `worktree <path>` lines, hence `LocationCompare::Lists`.
- **Why the probe contract exists** (PR #1 review): a plain `git status` runs the `core.fsmonitor` helper the repository configures and, through its background refresh, rewrites the index whose mtime is the activity signal; both were reproduced on git 2.54. `--no-optional-locks` stops the refresh; the `-c` overrides stop the helpers; the command line outranks every config file. `core.hooksPath=/dev/null` is belt and braces: no probe runs a hook-bearing command, but `worktree remove` is not a probe and the override costs nothing.
- **Why `--ignored=matching`** (PR #1 review): `status` lists no ignored path by default, and `git worktree remove` without `--force` deleted an ignored `.env` and `state/local-only.txt` from a clean worktree in the reviewer's fixture. `matching` lists each ignored path rather than collapsing ignored directories, which the per-path accounting needs; on a worktree with a large ignored tree (`node_modules`) the listing is long, and the accounting short-circuits once a path is inside a claimed `delete` finding.
- **Why an explicit surviving-ref set** (PR #1 review): `rev-list --all` includes `HEAD`, and traversal includes other worktrees' `HEAD`s, so both `--all` sketches of the first draft counted a unique commit as preserved. Naming the survivors and the candidate's commit by hash makes the answer independent of where the probe runs and covers a detached `HEAD`, whose commit nothing else holds once the worktree is gone.
- **Why no `prune`** (PR #1 review): `git worktree prune --dry-run --verbose` selected another, moved worktree's administrative directory in the reviewer's fixture; `remove` already deletes its own `.git/worktrees/<name>`.
- **Why `submodules` is a condition** (PR #1 review): `git worktree remove` exits 128 with "working trees containing submodules cannot be moved or removed" for a clean worktree with an initialized submodule, and `--force` is forbidden; reporting it as reclaimable would promise what the executor cannot do.
- `git status` on a large tree can take seconds; the probes run only for candidates that are unlocked, not in use and past the idle window, and in parallel. The 30 s timeout is the existing probe timeout in `exec/native.rs`.
- Claude Code's docs say it writes a marker into the git metadata of every worktree it creates. If the marker is a file in `<gitdir>`, it is the preferred signal, because a `WorktreeCreate` hook can place worktrees elsewhere; the path under `.claude/worktrees/` stays as the fallback.

## Dependencies

- [[adr/agent-footprint-worktrees-harness-data-and-bounded-consent-for-agents|Agent footprint: worktrees, harness data and bounded consent for agents]]
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]
- [[epics/DDC-001/stories/story-detection-rule-catalog|Detection rule catalog]]
- [[epics/DDC-001/stories/story-safe-cleanup-ddc-clean|Safe cleanup: ddc clean]]

## Attachments

- Claude Code worktrees: https://code.claude.com/docs/en/worktrees

---

*Last updated: 2026-10-18*

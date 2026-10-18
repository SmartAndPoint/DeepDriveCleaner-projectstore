---
type: story
id: "story-worktree-findings-and-provenance"
epic: "DDC-003"
title: "Worktree findings and provenance"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-18
updated: 2026-10-18
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

A linked git worktree becomes a finding. This story is §1 and §2 of the ADR "Agent footprint: worktrees, harness data and bounded consent for agents": the `git.worktree` and `git.worktree-orphan` rules, the `origin` provenance, the `worktree_of` grouping, the conditions under which a harness's worktree is `rebuildable`, the `contained` plan rule, and removal through `git worktree remove`. After it, `ddc scan` shows an agent's dead worktrees under their project and `ddc clean --yes --class rebuildable` removes them through git. Nothing agent-facing is added yet; `footprint` and `--agent` are separate stories.

The rule fires for every linked worktree in the scan roots. Only a harness's worktree can be `rebuildable`; a person's own stays `review`.

## Decomposition

Rough until the plan gate.

- [ ] Model: `Finding.origin: Option<Origin { harness, kind }>` and `Project.worktree_of: Option<PathBuf>`, both optional, `schema_version` unchanged; `docs/scan.schema.json` regenerated
- [ ] Matcher `Worktree`: `.git` is a file with `gitdir:`; the gitdir exists → `git.worktree`, else `git.worktree-orphan`; the main checkout from `<gitdir>/commondir`. Checked in `after_read` before the project-manifest exclusion, because a worktree is a project root and must still be claimed as a **region**: rules keep firing inside it, findings inside get `project.root = worktree` and `project.worktree_of = main`
- [ ] Provenance: `origin.harness = claude-code` when the path is under `<main>/.claude/worktrees/` or the gitdir carries the marker Claude Code writes (decided at the plan gate by reading a real worktree's gitdir on the reference machine); otherwise `git`
- [ ] Classification (`rules/classify.rs`), for `git.worktree` with a known harness only, in this order, stopping at the first failure with a `Note`: not locked (`<gitdir>/locked` absent) and not in use (the existing process check) → idle for `--worktree-min-age` (default 2 days; the worktree's own `index`, `HEAD`, manifests and lock files) → clean (`git status --porcelain --untracked-files=all`, empty) → no own commits (`git rev-list` of `HEAD` not reachable from any other ref; exact invocation at the plan gate). All pass → `rebuildable`, else `review` with the note (`locked`, `in-use`, `active`, `dirty`, `own-commits`, `status-unknown` on error or a 30 s timeout). Probes run in parallel across candidate worktrees and only for candidates
- [ ] Plan (`plan.rs`): an item inside a selected `git.worktree` item is dropped with the new reason `contained`; the worktree's reclaimable bytes include them
- [ ] Removal: `Removal::Native` for `git.worktree` with `git -C <main> worktree remove <path>` then `git -C <main> worktree prune`, location probe `git -C <main> worktree list --porcelain` **lists** the path (a new `LocationCompare::Lists`, since the probe prints several lines), fallback `None` (report only). `NativeCommand` needs `{main}` and `{path}` substitutions or a small variant; no `--force`, ever. `git.worktree-orphan`: `trash`, `--permanent` → `delete`
- [ ] Executor re-check (`exec/guard.rs`, `Catalog::verify`): the clean, lock and own-commits conditions are re-run before the native command; a `git.worktree` item whose plan action is `delete` or `trash` is `action-not-allowed`; the "no `.git` at the top level" refusal stays for every other rule
- [ ] Hints: `git.worktree` → "removed with git worktree remove; branch <branch> is kept, git branch -d to drop it"; `review` notes name the reason
- [ ] Render: a worktree's artifacts are shown indented under the worktree, and the worktree under its main project
- [ ] `--worktree-min-age <days>` on `scan` and `clean`; `settings.json` key `worktree_min_age_days` once DDC-002's settings story lands (not a dependency: the flag is enough here)
- [ ] README: the worktree rule, the two-day window, `.ddc-keep` and `--worktree-min-age`

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

<!-- How do we know we're done? Each criterion testable. Checked only with
     evidence at close: - [x] <criterion> — evidence: <test | command | file:line> -->

- [ ] Test (a real `git` repository in a temp tree, `TreeBuilder` plus `git init`/`git worktree add`): a worktree under `<main>/.claude/worktrees/x` is a `git.worktree` finding with `origin.harness = claude-code`; one made with `git worktree add ../y` is `git.worktree` with `origin.harness = git`; a copied directory whose `.git` file points to a missing gitdir is `git.worktree-orphan`; the main checkout itself is not a finding
- [ ] Test: a `.venv` with `pyvenv.cfg` and `uv.lock` inside the harness worktree is a `python.venv` finding with `project.root` = the worktree and `project.worktree_of` = the main checkout; `report::project::last_activity` for it reads the worktree's gitdir, not the main checkout's
- [ ] Test, one case per condition, each leaving every other condition satisfied: a harness worktree that is clean, unlocked, three days idle and without own commits is `rebuildable` and `selected`; dirty (one untracked file) → `review` with note `dirty`; one local commit not on any other branch → `own-commits`; `<gitdir>/locked` present → `locked`; a process whose cwd is inside it (the fake `ProcessTable`) → `in-use`; touched one hour ago → `active`; a `git` on `PATH` that hangs (a fake script) → `status-unknown` within the timeout; `origin.harness = git` → `review` with no probe run at all (the fake `CommandRunner` records no call)
- [ ] Test: a merged branch (its commits are on `main`, its remote deleted) passes the own-commits condition; a submodule checkout with changes inside the worktree makes it `dirty`
- [ ] Test (`plan.rs`): with the worktree selected, its inner `.venv` item is dropped as `contained` and the plan's total counts the worktree's bytes once; with the worktree `review`, the inner `.venv` is planned on its own as today
- [ ] Test (`clean_safety.rs`): removing a selected worktree runs exactly `git -C <main> worktree remove <path>` and `git -C <main> worktree prune` through the `CommandRunner`, never with `--force`; afterwards `git worktree list` does not list it, `.git/worktrees/<name>` is gone, the branch `worktree-<name>` still exists, and the journal line carries the command's tail. With the location probe not listing the path, the item is skipped `native-unavailable`-style with the location mismatch and nothing runs
- [ ] Test: a `git.worktree` item planned with action `delete` or `trash` is skipped `action-not-allowed`; a directory with `.git` at its top level that is **not** a linked worktree is still refused as before (the existing test in `clean_safety.rs` keeps passing)
- [ ] Test: the executor's re-check runs the clean and own-commits probes again; a worktree made dirty between scan and clean is skipped `changed-since-scan`
- [ ] Test (`scan_cli.rs`, `clean_cli.rs`): `--worktree-min-age 0` selects a clean harness worktree touched a minute ago; the default leaves it `active`; `--json` validates against the regenerated `docs/scan.schema.json`, and the committed schema equals the generated one
- [ ] On the reference machine: `ddc scan ~/Projects` lists every linked worktree `git worktree list` reports for the projects there, and the selected ones are exactly those a manual check (`git status`, `git log <branch> --not --all`, `ls .git/worktrees/*/locked`) agrees are dead; result in the final summary with counts and bytes
- [ ] `make check` passes

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- `report::project::git_dir` already follows `gitdir:` for activity; the matcher reuses it and adds `commondir`.
- The catalog rule "a directory with `.git` at its top level is never claimed" (story "Detection rule catalog") needs one carve-out: a **file** `.git` with `gitdir:` into `<x>/.git/worktrees/` is the worktree rule's marker. A directory `.git`, or a file `.git` pointing anywhere else (a submodule's `gitdir: ../.git/modules/…`), keeps today's behaviour: not claimed. A submodule checkout is therefore never a `git.worktree`.
- The executor's "not a project root" and "no `.git` at the top level" refusals (safety ADR §5) apply to `delete`/`trash`; `git.worktree` uses `Native` with `fallback` report-only, so the refusals are never reached for it, and the test for `delete`/`trash` on a worktree asserts the executor's `action_allowed` check instead.
- `NativeCommand` today substitutes only `{name}`; the worktree command needs the main checkout and the full path. Either add `{main}` and `{path}` substitutions or a `NativeCommand::worktree` variant; the plan gate decides. `LocationProbe` compares one path from stdout; `git worktree list --porcelain` prints `worktree <path>` lines, hence `LocationCompare::Lists`.
- `git status` on a large tree can take seconds; the probes run only for candidates that are unlocked, not in use and past the idle window, and in parallel. The 30 s timeout is the existing probe timeout in `exec/native.rs`.
- `git rev-list <HEAD> --not --all` counts the worktree's own branch among `--all`, so the branch must be excluded (`--exclude=refs/heads/<branch>` before `--all`, or `--not $(git for-each-ref --format=%(refname) | grep -v <branch>)`); verify on git 2.40+ at the plan gate.
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

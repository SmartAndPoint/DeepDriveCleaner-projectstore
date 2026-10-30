---
type: story
id: "story-main-only-through-pull-requests-in-the-code-and-the-vault"
epic: "DDC-005"
title: "Main only through pull requests, in the code and the vault"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-30
updated: 2026-10-30
external_refs: {}
tags: [ci, git, github, release]
code_refs: [".github/workflows/gate.yml", ".github/workflows/install.yml", ".github/workflows/release.yml", "scripts/ci-green.sh", "scripts/tag-release.sh", "scripts/publish-release.sh", "Makefile", "README.md", "AGENTS.md", ".githooks/pre-push"]
specs: []
started_at: 2026-10-30
closed_at: "2026-10-30T14:03:46.173Z"
plan_updated_at: null
---

# Main only through pull requests, in the code and the vault

| Field | Value |
|---|---|
| **Epic** | [DDC-005](../epic.md) |
| **Status** | in-progress |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

Until 2026-10-30, every session committed straight to `main` in both repositories, `SmartAndPoint/DeepDriveCleaner` and `SmartAndPoint/DeepDriveCleaner-projectstore` (`git push origin HEAD:main`). Neither repository protected `main`.

That had three costs:
- **A red commit can land on `main`.** Only the local pre-push gate stood in the way. On 2026-10-30, 3939793 passed it and then went red in CI.
- **Two sessions race for `main`.** The second push is refused, rebased, and gated again. To avoid that, the sessions took turns and waited for each other.
- **Nothing passes a check before it is on `main`.**

On 2026-10-30 the owner ruled: no direct push to `main`, in either repository, "the right way".

The change, as a GitHub repository ruleset on `main` in each repository, with no bypass actors:
- no direct push: a pull request is required, with 0 approvals (one maintainer, and GitHub refuses a self-approval);
- no force push and no deletion;
- linear history. Rebase merge is the only merge method, so every commit keeps its own message and its `Maintainer:` trailer.
- **Code repository only:** the Gate's `scope`, `gate`, `coverage`, `app` and `release-build` checks are required, from GitHub Actions. The branch does not have to be up to date with `main` first.
  - A job the scope rule skipped reports success, so a GUI-only PR still merges in about 2 minutes.
  - `main`'s own push run after the merge, Nightly and `make tag`'s `ci-green --full` judge the combination.
- Auto-merge on, and the head branch deleted on merge.

**In the code repository**, each session works on a branch started from a fresh `origin/main`:
1. `git push -u origin <branch>`; the pre-push hook gates it as before.
2. `gh pr create --fill`.
3. `gh pr merge --auto --rebase`.

**In the vault** there is one checkout on `main`, shared by every session, often with another session's uncommitted edits in it. Its recipe, in the vault's own `AGENTS.md`, never switches that checkout's branch and never resets it. One **re-sync** runs before a commit and after a merge:
- `origin/main` only ahead (a PR merged from another clone): `git merge --ff-only origin/main`. It rewrites only the files those commits changed, keeps staged and untracked work, and refuses rather than overwrites an uncommitted edit to one of them. Tested in a scratch repository.
- Same tree, other SHAs (the session's own rebase-merged PR): `git update-ref refs/heads/main origin/main <old>`, which touches neither the index nor the files. It compares trees, so a local commit that changes nothing is dropped to the reflog.
- Diverged: `git cherry -v origin/main HEAD` shows which side holds what, and the owner decides.

The steps:
1. Re-sync, then commit on `main`.
2. `git cherry -v origin/main HEAD`: the PR carries every local commit ahead of `origin/main`, whoever made it. A `-` line is already merged under a new SHA, so re-sync first. No `+` line means another session's PR already carried it.
3. `git push origin HEAD:refs/heads/<topic>`.
4. `gh pr create --base main --head <topic> --title … --body …` from the last commit. `--fill` fails: with no local `<topic>` branch it cannot resolve `origin/main...<topic>`.
5. `gh pr merge <n> --rebase`, with no `--auto`: with no checks, auto-merge can be refused while GitHub is still computing mergeability.
6. Re-sync.

Any other branch (a conflicting PR to rebase, such as the open vault PR #1) is worked in a throwaway `git worktree`, never in the shared checkout.

Not `git pull --rebase --autostash`. Its autostash runs a `reset --hard` and a checkout, about 0.6 s on this vault. A write landing in that window (another session's edit, a regenerated `kanban.md`) stops the rebase half-way: the other session's edits are gone from the files and are kept only in a dangling autostash commit, outside `git stash list`. The critic reproduced it in a scratch repository. The autostash also drops what was staged.

**In both, a step is done when its PR is merged** (`gh pr view <n> --json state` is `MERGED`), not when auto-merge is on. A red check, or a conflict with a newer `main`, leaves auto-merge waiting or stops it, with only an email. The wait is `gh pr checks <n> --watch --fail-fast`.

**Never `--delete-branch`.** On a PR that merges at once, gh also tries to delete the local branch, which a session may still be standing in. GitHub deletes the remote branch (`delete_branch_on_merge`).

In the code repository nobody waits for another session's push. In the vault, one session's PR may carry another's commits; step 2 shows that before the push.

**The rule is written down for every session and agent**, in `AGENTS.md` of both repositories; the vault gets a `CLAUDE.md` that imports it. The owner's ruling: work in either repository goes through a PR, and the session that opened a PR may merge it when it has no conflicts.

**The Gate runs only on push.** `gate.yml` drops its `pull_request` trigger.
- Every PR here comes from a branch of the same repository, and that branch's push run already posts the checks on the PR's head commit.
- A `pull_request` run would post a second set under the same names. Its `scope` answers `rust` for a pull request, so a GUI PR would run all of Rust again, and two runs would race for the same required check.
- A PR from a fork gets no Gate run, so it cannot merge. A maintainer pushes its commits to a branch here.

**The release stays as it is.**
- `make tag` and publish still require `main` with `HEAD` equal to `origin/main`.
- The version bump goes in through a PR. Then `git switch main && git pull --ff-only`, and `make tag`.
- Tags are not covered by the ruleset, so `make tag` still pushes its tag.

## Decomposition

- [x] Ruleset "main through pull requests" on `SmartAndPoint/DeepDriveCleaner`, created through `gh api`: deletion, non_fast_forward, required_linear_history, pull_request (0 approvals, merge method rebase), and required_status_checks for `scope`, `gate`, `coverage`, `app` and `release-build` with the GitHub Actions integration. Repository settings: `allow_auto_merge`, `delete_branch_on_merge`, and rebase merge only.
- [x] The same ruleset without required checks on `SmartAndPoint/DeepDriveCleaner-projectstore`, with the same settings.
- [x] `gate.yml` and `install.yml`: drop `pull_request`. Their concurrency groups and comments no longer name it.
- [x] `scripts/ci-green.sh`: "no push run yet" means "main's run starts within a minute after a merge", not "push it".
- [x] `scripts/tag-release.sh` and `scripts/publish-release.sh`: "push first" becomes "merge the PR and pull".
- [x] The code repository's AGENTS.md, a section outside the projectstore block, and the README's development section:
  - the branch → PR → auto-merge flow, binding every session and agent;
  - merge without asking when there are no conflicts;
  - a pointer to the vault's recipe;
  - the merged check;
  - never `--delete-branch`;
  - never push to `main`.
- [x] The vault's own `AGENTS.md` with the recipe above, and a `CLAUDE.md` that imports it.
- [x] Vault runbook "Cutting a release: tag, CI, archives, dist and publish": the bump goes in through a PR, then pull `main`.
- [x] The code repository's pre-push hook refuses a push to `main` before it runs the gate.
- [x] Turned on only after the other session's in-flight push to `main` lands, and that session is told the new flow.

## Implementation Plan

1. Write this story and critique it. The first critique asked for: the shared-vault recipe; the merged check; no `--auto` in the vault; `install.yml`; the `/sp` commands; a stronger AC 5.
2. Once the other session's push has landed, create both rulesets and the repository settings.
3. Make the code changes on branch `main-through-prs`, as this story's first PR. It proves the flow.
4. The vault's `AGENTS.md` and `CLAUDE.md` through vault PRs. The runbook goes in with this story's close, also through a PR. (The vault's first PR, #2, was another session's close.)

## Acceptance Criteria

- [x] `git push origin HEAD:main` is refused in both repositories. In the vault the server refused an empty-commit push (GH013). In the code repository the pre-push hook refuses it before the gate, and the same ruleset stands behind the hook; showing the server refusal there would take `--no-verify`, which AGENTS.md forbids, so this criterion was reworded at close. — evidence: `gh api repos/SmartAndPoint/DeepDriveCleaner-projectstore/rules/branches/main` and `.../DeepDriveCleaner/rules/branches/main` (deletion, non_fast_forward, required_linear_history, pull_request, plus required_status_checks in the code repository); rulesets 26206450 and 26206449 `current_user_can_bypass: never`; `.githooks/pre-push:10-16` at 72c3765 exits 1 with "main takes changes only through a pull request" for a `refs/heads/main` line and for its `(delete)` (reviewer, scratch repository)
- [x] In the code repository, a PR cannot merge while a required check is pending or red, and auto-merge merges it once the checks pass. Shown by this story's PR: `autoMergeRequest` set while a run is pending, then `state: MERGED`. — evidence: `gh pr view 3 --json autoMergeRequest,mergedAt,statusCheckRollup`: auto-merge (rebase) on at 13:35:36Z with every check in progress; the last required check (`gate`) at 13:44:45Z; merged at 13:45:28Z as 72c3765
- [x] A GUI-only PR merges on the skipped Rust jobs, with no `pull_request` run, in under 5 minutes of jobs. Shown by the first such PR after this story; until then this box stays open. — evidence: PR #5 "feat: Show in Finder for a coding agent's folders in the Whole disk view" (app sources and DDCKit only), merged 14:01:19Z; its only Gate run, 41067221767, is `event=push`: `scope` 13:54:57–13:55:06 and `app` 13:54:59–13:57:13 succeeded, `coverage`, `release-build`, `check` and `gate` skipped; 2 min 20 s of jobs
- [x] A vault PR merges with `gh pr merge --rebase`. — evidence: vault PRs #3, #4 and #5 MERGED (13:11:31Z, 13:12:37Z, 13:26:56Z); `git log origin/main` linear, no merge commits; ruleset `allowed_merge_methods: ["rebase"]`
- [x] `gh api repos/SmartAndPoint/<repo>/rulesets/<id>` shows the following, for each repository:
  - `enforcement: active`, `target: branch`, and `~DEFAULT_BRANCH` in `conditions.ref_name.include`;
  - an empty `bypass_actors`, and the rules above;
  - in the code repository, `strict_required_status_checks_policy: false` and the five contexts at `integration_id` 15368;
  - `require_extra_approval_for_unattributed_changes: true` is GitHub's default and does nothing at 0 required approvals.
  — evidence: `gh api repos/SmartAndPoint/DeepDriveCleaner/rulesets/26206449` and `gh api repos/SmartAndPoint/DeepDriveCleaner-projectstore/rulesets/26206450` (reviewer, 2026-10-30)
- [x] `gh api repos/SmartAndPoint/<repo> --jq '[.allow_merge_commit,.allow_squash_merge,.allow_rebase_merge,.allow_auto_merge,.delete_branch_on_merge]'` is `[false,false,true,true,true]` for each repository. — evidence: that command, both repositories (reviewer, 2026-10-30)
- [x] A vault PR re-syncs the shared checkout with `update-ref`, leaving `git status --porcelain` unchanged. — evidence: vault `git reflog main`: "re-sync after merge" to 228c2a9 and to 23b60fb, each onto an equal tree (d80690c, eb4d698); `update-ref` touches neither the index nor the files
- [x] Owner checks: none. — evidence: every criterion above is an API read or a recorded run

## Final Summary

**What changed.**
- `main` in both repositories is behind a ruleset: no direct push, no deletion, no force, linear history, pull requests merged by rebase only, and no bypass actors. In the code repository the merge also waits for the Gate's `scope`, `gate`, `coverage`, `app` and `release-build`. Auto-merge is on, and merged branches are deleted on GitHub.
- The Gate and the install check run on push only. Every PR's branch push runs them, and `main` runs them again after the merge.
- The release scripts' messages say "merge the PR and pull" where they said "push first". The pre-push hook refuses `main` in a second, before minutes of gate.
- AGENTS.md in both repositories binds every session and agent to the flow. The vault, one checkout shared by sessions, gets its own recipe: commit on `main`, push a topic, merge its PR, then re-sync with a fast-forward or `update-ref`, never a reset or an autostash pull.
- The release runbook puts the version bump through a PR, then `git switch main && git pull --ff-only` before `make tag`.

**Why.** A red commit had reached `main`, and two sessions raced for it and waited on each other. The owner asked for no direct merge into `main` in either repository.

**Tests executed.** The rulesets and settings read back through `gh api`. Code PR #3 was this story's own: blocked while its checks ran, then merged by auto-merge 43 s after the last one. Code PR #5 merged on skipped Rust jobs. Vault PRs #3, #4 and #5 merged, and the checkout re-synced without a reset. The hook's guard was run in a scratch repository.

**Risks and follow-ups.**
- If Actions is down, nothing merges in the code repository until the owner edits the ruleset.
- The hook's guard has no automated test: nothing runs `.githooks/pre-push` itself.
- A clone still on an older `main` runs the old hook, which has no guard, until it is fast-forwarded.
- Auto-merge once stayed at CLEAN with every check done (PR #5). It was merged by hand, which this story's rule allows.

## Technical Notes

- **Skipped jobs and required checks.** A job skipped by its `if` reports success. That is what lets a GUI-only PR merge. It is safe here because `scope` is required too, and `scope` decides the skip. A failed `scope` runs the Rust jobs (`!cancelled()`).
- **Why the branch need not be up to date.** Requiring it would make every parallel PR rebase and re-run CI after each merge, which is the waiting this story removes. A merge queue would test the combination before the merge. It needs a `merge_group` trigger and its own scope rule, and is a follow-up if `main` goes red after a merge.
- **No bypass, the owner included.** In an emergency the owner edits the ruleset, which is an admin action that leaves an audit entry, instead of pushing past it.
- **The owner's `/sp:commit` and `/sp:push` commands** live outside these repositories, in `~/.claude/commands/sp/`. Their "commit and push" now fails on `main`, so AGENTS.md is what a session follows. Adapting them is the owner's.
- **Lock-out.** If Actions is down, nothing merges in the code repository until the owner edits the ruleset. A bypass actor with `bypass_mode: pull_request` (merge past checks, still no direct push) was considered and left out: the owner asked for no direct merge.
- **The scope window.** Branch runs now outnumber `main`'s. If `main`'s last Rust-green run falls outside the 50 runs `gate-scope.sh` reads, the scope answers `rust`. That is safe, only slower.
- **The pre-push hook stays.** It catches a red branch before CI does. Making it lighter, since CI gates the merge, is a separate decision.

## Dependencies

- Story "Gate by what changed: load at night, one gate per release, app-only pushes skip Rust" (DDC-005): the `scope` rule the required checks rely on.
- Story "Move OS calls behind the platform module" (DDC-007), whose push is in flight in another session. The rulesets are turned on after it lands.

## Attachments

-

---

*Last updated: 2026-10-30*

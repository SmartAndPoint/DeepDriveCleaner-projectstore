# Working in this vault

This repository is the ProjectStore vault of DeepDriveCleaner
(`SmartAndPoint/DeepDriveCleaner`). The rules below bind every session and every agent,
human-started or spawned.

## Every change goes through a pull request

- `main` takes no direct push, here or in the code repository: a ruleset requires a
  pull request and refuses `git push origin HEAD:main`. This holds for every session and
  every agent, whatever it is changing.
- A pull request that merges without conflicts may be merged by the session that opened
  it. One with conflicts is rebased onto `origin/main` first; never force it through.
- This checkout is on `main`, shared by every session, often with another session's
  uncommitted edits in it. Never switch its branch, never `git reset --hard` it, and never
  `git pull --rebase --autostash` in it: the autostash hard-resets the files, and a write
  landing in that moment leaves another session's edits only in a dangling autostash.
- **Re-sync** moves local `main` to `origin/main` without a reset. Run it before you
  commit and after your PR merges:

  ```sh
  git fetch origin main
  old=$(git rev-parse HEAD)
  if git merge-base --is-ancestor "$old" origin/main; then
    git merge --ff-only origin/main
  elif [ "$(git rev-parse "$old^{tree}")" = "$(git rev-parse 'origin/main^{tree}')" ]; then
    git update-ref -m "re-sync after merge" refs/heads/main origin/main "$old"
  else
    echo "diverged: see git cherry -v origin/main HEAD; do not rebase here"
  fi
  ```

  - `origin/main` only ahead (another clone's PR merged): the fast-forward rewrites only
    the files those commits changed, and refuses rather than overwrites an uncommitted
    edit to one of them; staged and untracked work is kept. If it refuses, that file is the owner's call.
  - Same tree, other SHAs (your rebase-merged PR): `update-ref` moves the ref and touches
    neither the index nor the files. It checks the tree, not the commits, so a local
    commit that changes nothing (an empty commit, a commit and its revert) is dropped to
    the reflog.
  - Diverged: `git cherry -v origin/main HEAD` marks with `-` the local commits already
    on `main` under new SHAs and with `+` the ones that are not. Ask the owner.
- The steps:
  1. Re-sync, then commit on `main`, listing your files; never `git add -A`.
  2. `git cherry -v origin/main HEAD` lists what the PR will carry: every local commit
     ahead of `origin/main`, whoever made it. A `-` line means a merged PR has not been
     re-synced: re-sync first. No `+` line means another session's PR already carried
     yours: stop. If a `+` line is another session's commit, tell that session it is
     going out with yours. Then `git push origin HEAD:refs/heads/<topic>`.
  3. `gh pr create --base main --head <topic> --title "$(git log -1 --format=%s)"
     --body "$(git log -1 --format=%b)"`. `--fill` fails here: the topic branch exists
     only on GitHub.
  4. `gh pr merge <topic> --rebase`, with no `--auto`: there are no checks, and GitHub can
     refuse auto-merge while it is still computing mergeability. Retry once.
     Never pass `--delete-branch`; GitHub deletes the remote branch.
  5. Re-sync.
- Any branch other than `main` (a conflicting PR to rebase, a topic to fix) is worked in
  a throwaway worktree, never in this checkout: `git fetch origin <topic> main`,
  `d=$(mktemp -d) && git worktree add "$d" origin/<topic>`, rebase onto `origin/main` there, `git -C "$d" push --force-with-lease
  origin HEAD:refs/heads/<topic>`, then `git worktree remove "$d"`.
- If an autostash pull was run here anyway and printed `Created autostash: <sha>` before
  failing, do not `git rebase --abort` (it overwrites the files again): stop, tell the
  owner and the other sessions; `<sha>` holds the stranded edits.
- A change is done when `gh pr view <topic> --json state` says `MERGED`.

## Derived views

`kanban.md`, `code-map.md`, `graph.md` and the folders' `README.md` indexes are
regenerated from the whole working tree, untracked files included. Commit a regenerated
view only in the pull request that also commits every artifact it lists, so the public
board never names a file the repository does not hold.

## The code repository's rules

`AGENTS.md` in `SmartAndPoint/DeepDriveCleaner` holds the rest: the gate, the
release, and the ProjectStore agents to run before and after an artifact or a change.

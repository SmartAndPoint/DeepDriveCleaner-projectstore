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
  uncommitted edits in it. Never switch its branch and never `git reset --hard` it.
  1. Commit on `main`, listing your files; never `git add -A`.
  2. `git fetch origin main && git log --oneline origin/main..HEAD` lists what the PR
     will carry: every local commit ahead of `origin/main`, whoever made it. Empty means
     another session's PR already carried yours: stop, there is nothing to open. If it
     lists another session's commit, tell that session it is going out with yours.
     Then `git push origin HEAD:refs/heads/<topic>`.
  3. `gh pr create --base main --head <topic> --title "$(git log -1 --format=%s)"
     --body "$(git log -1 --format=%b)"`. `--fill` fails here: the topic branch exists
     only on GitHub.
  4. `gh pr merge <topic> --rebase`, with no `--auto`: there are no checks, and GitHub can
     refuse auto-merge on a clean PR. Retry once if it is still computing mergeability.
     Never pass `--delete-branch`; GitHub deletes the remote branch.
  5. Move local `main` onto the merged commits without touching the index or the files.
     The rebase merge gave them new SHAs but the same tree, so nothing on disk changes:

     ```sh
     git fetch origin main
     old=$(git rev-parse HEAD)
     if [ "$(git rev-parse "$old^{tree}")" = "$(git rev-parse 'origin/main^{tree}')" ]; then
       git update-ref -m "re-sync after merge" refs/heads/main origin/main "$old"
     else
       echo "origin/main holds a change this checkout lacks: do not rebase here"
     fi
     ```

     Never `git pull --rebase --autostash` in this checkout while another session is
     working: it hard-resets the files, and a write landing in that moment leaves the
     other session's edits only in a dangling autostash. When the trees differ, ask the
     owner. If such a pull was run and printed `Created autostash: <sha>` before failing,
     do not `git rebase --abort`: run `git rebase --continue`, then
     `git stash apply <sha>`, and tell the other sessions.
- A change is done when `gh pr view <topic> --json state` says `MERGED`.

## Derived views

`kanban.md`, `code-map.md`, `graph.md` and the folders' `README.md` indexes are
regenerated from the whole working tree, untracked files included. Commit a regenerated
view only in the pull request that also commits every artifact it lists, so the public
board never names a file the repository does not hold.

## The code repository's rules

`AGENTS.md` in `SmartAndPoint/DeepDriveCleaner` holds the rest: the gate, the
release, and the ProjectStore agents to run before and after an artifact or a change.

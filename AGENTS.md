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
  2. `git push origin HEAD:refs/heads/<topic>`.
  3. `gh pr create --head <topic> --fill`.
  4. `gh pr merge <topic> --rebase`, with no `--auto`: there are no checks, and GitHub can
     refuse auto-merge on a clean PR. Retry once if it is still computing mergeability.
     Never pass `--delete-branch`; GitHub deletes the remote branch.
  5. `git pull --rebase --autostash origin main`. The rebase merge gave your commits new
     SHAs; the pull replaces your local copies with them and puts back the other
     sessions' edits.
- A change is done when `gh pr view <topic> --json state` says `MERGED`.

## Derived views

`kanban.md`, `code-map.md`, `graph.md` and the folders' `README.md` indexes are
regenerated from the whole working tree, untracked files included. Commit a regenerated
view only in the pull request that also commits every artifact it lists, so the public
board never names a file the repository does not hold.

## The code repository's rules

`AGENTS.md` in `SmartAndPoint/DeepDriveCleaner` holds the rest: the gate, the
release, and the ProjectStore agents to run before and after an artifact or a change.

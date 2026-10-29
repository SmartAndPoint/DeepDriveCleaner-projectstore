---
type: adr
id: "cleanup-safety-model-risk-classes-plan-and-confirmation"
title: "Cleanup safety model: risk classes, plan and confirmation"
status: accepted
date: 2026-10-17
authors: ["Evgenii Konev"]
tags: [safety, cleanup, risk]
external_refs: {}
supersedes: null
superseded_by: null
review_status: reviewed
reviewed_at: 2026-10-28
---

# Cleanup safety model: risk classes, plan and confirmation

| Field | Value |
|---|---|
| **Status** | accepted |
| **Date** | 2026-10-17 |
| **Authors** | Evgenii Konev |

---

## Context

`ddc` deletes gigabytes in a developer's home directory, and a mistake there is expensive: a Docker VM image with database volumes, a `build/` that turned out to hold experiment results, or the venv of a project that is being worked on right now. The research [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]] showed:

- the same names mean different things (`env/`, `build/`, `dist/`), and next to `*.xcodeproj` there can be an unrecoverable `build/`;
- the cost of recovery varies: from zero (`__pycache__`) to gigabytes of downloads (Ollama models, toolchains) and unrecoverable data (photos, iPhone backups, volumes in VM images);
- `CACHEDIR.TAG` is present both in venvs created by uv and in `target/` created by cargo. Per the specification, it is a hint for backups, not permission to delete automatically;
- many global caches have a native cleanup command, and it is safer than `rm -rf`;
- moving to the Trash on the same volume is an instant `rename(2)`, but the space is freed only after the Trash is emptied.

The user decided (2026-10-17) that background mode only scans and notifies, and deletes nothing.

## Decision

### 1. Four risk classes and a removal method per rule

Every rule in the rule catalog declares a risk class and a removal method. The class determines default selection and whether `--yes` is allowed; the method determines how the executor removes.

| Class | What it is | Example | In the plan by default | `--yes` |
|---|---|---|---|---|
| `safe` | cache, restored automatically | `__pycache__`, `~/.npm/_cacache`, `~/Library/Caches/pip` | selected | yes |
| `rebuildable` | project artifact: an environment restored by installing from a lock file, or a build product restored by building | `.venv` with `pyvenv.cfg` and `uv.lock`, `node_modules` with `package-lock.json`, `target/` next to `Cargo.toml` | selected only under the conditions of item 3 | only with an explicit `--class rebuildable` |
| `review` | expensive to restore, or not certain to be restorable | ML models, toolchain versions, data of old IDEs, artifacts without a lock file, `build/` without a reliable marker | **not selected** | no, interactive only |
| `user-data` | user data | photos, videos, backups, downloads, messengers, cloud folders | **never enters the plan** (amended 2026-10-20: an entry the user picks, to the Trash only) | — |

Removal methods:

| Method | What it does | Where it is used |
|---|---|---|
| `native` | the tool's native command | global caches that have one |
| `delete` | directory removal via a tombstone (item 5) | `safe`, `rebuildable` with effective class `rebuildable` |
| `trash` | move to the Trash (`NSFileManager.trashItem`, no AppleScript) | `review` by default, including `rebuildable` downgraded to `review` (item 3); `--permanent` replaces it with `delete` |
| `native-only` | **only** the native command, and only when the daemon is running; otherwise a report line with instructions. Only commands that delete neither containers nor volumes are allowed, and only with an explicit target: `docker --context desktop-linux builder prune -f`, `docker --context desktop-linux image prune -f` (no `-a`; if that context does not exist, instructions only), `podman image prune -f`. Space is freed inside the VM image (`Docker.raw`), and `statfs` may not change until the runtime shrinks the image; the plan line says so. `system prune` of any runtime is forbidden: it deletes stopped containers together with their data | VMs and stateful containers: Docker Desktop, podman, colima, lima, OrbStack, Rancher Desktop |
| `none` | report only | `user-data` (amended 2026-10-20: picked entries go to the Trash), Trash |

For `native-only`, removing a directory or a file is forbidden: the executor itself checks the plan's action against the rule's method and refuses (`action-not-allowed`) if the plan asks for something the rule does not allow — `delete`/`trash` for `native-only`, any action for `user-data` and `none` (amended 2026-10-20: `trash` for a picked entry of the five user-data folders, see Amendments). The plan is not treated as trusted input: a GUI or another client of the core can build it. The plan line for a native command names the command and what it deletes.

### 2. Rule priority

The walk goes top-down; for each directory, the decision is made in order, and the first match wins:

1. exclusions: `--exclude PATH`, a `.ddc-keep` file in the directory or in any of its ancestors inside the scan root → a finding with class `user-data` and the label "protected by user". The presence of `<dir>/.ddc-keep` is checked before each directory is read (`faccessat`), so a marker inside a known path (`~/.cache/uv/.ddc-keep`) also takes effect. If `.ddc-keep` is found anywhere inside an already claimed subtree (`node_modules/foo/.ddc-keep`), the whole finding becomes `user-data` "protected by user" (amended 2026-10-20 for the user's own folders: see Amendments): partial removal is not supported; cloud regions (`~/Library/CloudStorage`, `~/Library/Mobile Documents`) and dataless directories are not walked;
2. known-path rules: an exact match of the absolute path (`~/.cache/uv`, `~/Library/Caches/pip`, …), only inside the scan roots. They are checked before the directory is read, so they do not trigger TCC prompts in other apps' regions. An exact path is more specific than any name-based rule;
3. a directory that is an **ancestor** of a path from the known-path rules (for example, `~/.cargo/registry` for `~/.cargo/registry/cache`) is never claimed whole — it is only walked into;
4. specialized rules (directory name + context: `pyvenv.cfg`, `package.json` in the parent, `Cargo.toml`, etc.);
5. the generic `CACHEDIR.TAG` rule → no higher than `review`.

A matched directory claims its whole subtree: rules inside it are not applied. The exception is `~/Downloads`: it is a user-data "region" inside which rules keep working (a downloaded repository with `node_modules` will be found), and the remainder is attributed to a `user-data` finding.

### 3. Conditions for selecting `rebuildable` by default

A `rebuildable` item is selected in the plan only if all four conditions hold:

- the finding does not lie inside a top-level hidden directory of `$HOME` (`~/.vscode/extensions/…`, `~/.local/pipx/…`, `~/.local/share/nvim/lazy/…`, `~/.opencode/…`) or inside `~/Library`: there, project artifacts (environments and build products) belong to applications and tools, and such findings get `review`;
- for **environments** (venv, `node_modules`, `Pods`), the project has a manifest **and** a lock file (`uv.lock`/`poetry.lock`/`requirements*.txt`/`Pipfile.lock`; `package-lock.json`/`pnpm-lock.yaml`/`yarn.lock`/`bun.lock*`; `Podfile.lock`). Without a lock file, the rule downgrades the class to `review`. For **build products** (`target/`, Gradle `build/`, CMake build), a manifest is enough: their contents do not depend on a lock file;
- there is an unambiguous activity signal (item 4), and it is older than `--min-age` (14 days by default). No signal means the project is considered active;
- the finding's path is not in use by a running process (item 5; enforced at execution only, see the 0.2.1 amendment).

### 4. Project activity

The project root is the nearest ancestor of the finding that has `.git` (a directory, **or a file** for a worktree), otherwise the manifest's directory. Activity is the maximum mtime among: `<gitdir>/index` and `<gitdir>/HEAD` (for a worktree, `gitdir` is read from the `.git` file), and the project's manifests and lock files. If there is no signal at all, activity is unknown and the item is not selected.

### 5. Plan, confirmation and execution

- `ddc scan` only reads.
- `ddc clean` **always rescans** and builds the plan from the fresh scan, shows it (what, how much is allocated, how much it frees — the total is `~` (an estimate; `≥` on an individual finding is shown by `ddc scan`, where the private size is measured and noticeably smaller than allocated); for `native-only` the space is freed inside the VM image and is not included in the total — and how it is removed) and waits for confirmation. `--dry-run` prints the plan and exits.
- **Amended 2026-10-24** (story "Faster clean: re-check only what the selection needs", see Amendments): a clean of an explicit item list (`--items`) re-checks its items with a walk pruned to what their verdicts need the walk to have visited, and walks the whole home only when the list needs it. The terminal's `ddc clean` without `--items` always rescans, as above.
- `--class` is repeatable and works as a filter (`--class safe --class rebuildable`). `--yes`, non-interactively, removes only items **selected by default**, and only from the classes listed in `--class` (default `safe`). With `--yes`, class `review` in `--class` is not accepted (error). `review` and `native-only` always require interactive confirmation.
- The scan root itself is never removed: `ddc clean ~/.cache/uv` refuses; you need to scan the parent.
- A scan takes a shared file lock, a cleanup takes an exclusive one (`flock` on `<data-dir>/lock`). A background scan skips the run when the lock is held.
- The data directory (`last-scan.json`, the lock) and the log directory are set by `--data-dir` / `DDC_DATA_DIR`; by default `~/Library/Application Support/DeepDriveCleaner` and `~/Library/Logs/DeepDriveCleaner`. Tests always pass a temporary directory.

Before each action, the executor re-checks the finding (protection against TOCTOU and against "same path, different directory"; amended 2026-10-20: an entry of the user's own folders gets its own checks, see Amendments):

- the path exists and is a directory, not a symlink;
- the `(dev, ino)` of the finding and of its parent match the values recorded during the scan; otherwise skip with reason `changed-since-scan`;
- the rule's marker still matches;
- the path lies inside the scan root, is not a protected folder (item 6), has no `.git` at its top level and is not a project root;
- the finding was read completely: if it contained unreadable directories or another filesystem was mounted inside it, removal is rejected (`incomplete`);
- after the rename to a tombstone and before the Trash or the fallback, `(dev, ino)` is checked once more through the same parent descriptor (a tool probe can run between the check and the action);
- there is no `.ddc-keep` in the finding (at any depth) or in its ancestors inside the scan root;
- the path is not in use by a process: no process of the current user has, inside the finding, its executable path (the first string of the `sysctl(KERN_PROCARGS2)` buffer — the path passed to `execve`, before symlink resolution: `.venv/bin/python` is a symlink to an interpreter outside the venv), a command-line argument, its working directory (`proc_pidinfo(PROC_PIDVNODEPATHINFO)`) or a `VIRTUAL_ENV`/`CONDA_PREFIX` environment variable. A relative executable path and relative arguments are first joined with the process's working directory (`./.venv/bin/python app.py`). If a path is relative and the working directory cannot be determined, all plan findings whose name matches the first component of the relative path are blocked (`./.venv/bin/python` → all `.venv`), with reason `in-use-unknown-cwd`. Otherwise skip with reason `in-use` (amended 2026-10-20 for 0.2.1: open files count for `delete` and `native` items, `native-only` items are exempt; see Amendments);
- for rules with an owner process (for example, `Spotify` for its cache), that process is not running.

Removal (`delete`): the directory is first renamed to `<name>.ddc-deleting-<time>` with `renameat` relative to the parent descriptor already verified by `(dev, ino)` (a single operation), then removed without following symlinks. On `EACCES`, the executor adds `u+w` to directories inside the finding and retries. An interrupted removal leaves a tombstone, which the next scan finds as `safe`.

Before cleanup, local Time Machine snapshots are checked (`tmutil listlocalsnapshots /`). If there are any, the user gets a warning that the space will be freed only after the system deletes them.

The summary is shown separately: deleted, moved to the Trash, actually freed (by `statfs` before and after, with the caveat that APFS frees space asynchronously).

### 6. Protected folders

**The folders themselves** are protected, not their contents. The `delete`/`trash` executor never removes: `/`, `/Users`, `$HOME`, any scan root, `~/Documents`, `~/Desktop`, `~/Pictures`, `~/Movies`, `~/Music`, `~/Downloads`, `~/Projects`, `~/Library`, `~/Library/Caches`, `~/Library/Application Support`, `~/Library/Containers`, `~/Library/Group Containers`, `~/.config`, `~/.ssh`, `~/.gnupg`, `~/.local`, `~/.cache`, `/System`, `/Applications`, `/Library`, `/usr`, `/opt`, `/opt/homebrew`. Neither removed nor walked: `~/Library/Mobile Documents`, `~/Library/CloudStorage`. iPhone backups (`~/Library/Application Support/MobileSync/Backup`) are a `user-data` finding: the size is counted (Full Disk Access is required), rules inside are not applied. Native commands work inside their own regions and are not limited by folder protection; therefore the rule catalog admits only commands that act strictly within the finding (for example, `uv cache clean`, `go clean -modcache`), and only if the tool itself reports that path as its own: for caches, the path from the tool (`uv cache dir`, `go env GOMODCACHE`/`GOCACHE`) equals the finding's path; for toolchains, the root from the tool (`pyenv root`, `rustup show home`) equals the root that contains the finding; otherwise, the rule's fallback is used. The command runs with a 10-minute timeout. `brew cleanup`, which removes old formula versions in `/opt/homebrew`, is not used in v1: the Homebrew cache is removed as a directory. Amended 2026-10-28 (story "Whole disk: free space with the owning tool", see Amendments): `brew cleanup` runs only as the Homebrew area's action, from a person's confirmation, never as a rule of the catalog; the `cache.homebrew` rule still removes the cache as a directory. Amended 2026-10-29 (story "Whole disk: Conda environments and its package cache, with conda", see Amendments): conda is found among known installs and asked for its folders, conda may rewrite its own list of environments outside the finding, and a conda package cache has no fallback. Amended 2026-10-29 (story "Whole disk: Homebrew packages, their sizes, and brew uninstall", see Amendments): brew may also uninstall a formula a person picked, and the formulae that leaves orphaned, with `--formula`, after a re-read.

### 7. Log

Every action is a JSON line in `<log-dir>/actions.jsonl` with `schema_version`, time, path, `(dev, ino)`, rule, class, method, allocated size, private size (where it was measured), result, skip reason and the tail of the native command's output. Amended 2026-10-28 (see Amendments): the journal has its own schema version and a second kind of line, for an area action.

### 8. Background mode

A LaunchAgent runs `ddc scan --background` on a schedule. It saves the report to `<data-dir>/last-scan.json` and shows a notification if more than the threshold can be freed. The background path contains no call to the cleanup executor. Without Full Disk Access, the background scan does not open regions where macOS prompts the user (`~/Desktop`, `~/Documents`, `~/Downloads`, `~/Pictures`, `~/Movies`, `~/Music`, `~/Library/Containers`, `~/Library/Group Containers`, mail, messages, contacts, call history, calendars, reminders, iPhone backups): a read blocks until the prompt is answered, and the background process hangs (observed 2026-10-17). With Full Disk Access granted to the `deepdrivecleaner` binary, there are no prompts and everything is scanned. For an unsigned binary, the permission is tied to the build and is lost on update; this is solved by Developer ID signing.

## Rationale

1. The risk class separates "no need to think" from "need to think", and the user sees it in every line of the plan.
2. A lock file and known activity make `rebuildable` honest: without them, restoring can produce a different environment, or the project is still in progress.
3. Rule priority keeps a generic signal (`CACHEDIR.TAG`) from overriding a specialized rule with stricter conditions, and keeps an ancestor of a known path from swallowing it together with neighboring data.
4. `native-only` for VMs closes off the most expensive loss scenario: one plan line cannot wipe all the database volumes.
5. Comparing `(dev, ino)` during the re-check catches the case where a different project with the same name and marker now sits at the old location.
6. The tombstone makes an interrupted removal visible and finishable; without it, a half-deleted venv without `pyvenv.cfg` falls out of every rule.
7. The Trash is instant but does not free space, and the user's goal is space. So the Trash is used where undo matters (`review`), and `rebuildable`, restorable from a lock file, gets direct removal.

## Alternatives Considered

### Alternative A: background mode cleans `safe` automatically

**Pros**:
- Space is freed without the user's involvement.

**Cons**:
- Removal happens without the person knowing; a cache that is needed right now (for example, for an offline build) disappears.

**Rejected because**: the user chose the "scan only" mode. It can come back later as an option. The owner turned the option down again on 2026-10-28: an unattended clean would remove a project kept inside a path that looks like a cache, with nobody watching (story "Automatic cleaning of download caches after a scheduled scan", DDC-002, parked).

### Alternative B: everything to the Trash by default

**Pros**:
- Any action can be undone; a move on the same volume is instant.

**Cons**:
- Space is not freed until the Trash is emptied, and the user cleans the disk precisely because space ran out — they will empty the Trash as the next step, and the undo will be gone anyway.

**Rejected because**: it contradicts the goal for bulk `safe`/`rebuildable`. Accepted for `review`.

### Alternative C: delete everything immediately, without classes

**Pros**:
- Simpler implementation.

**Cons**:
- No protection against removing expensive or unrecoverable data.

**Rejected because**: unacceptable risk to user data.

### Alternative D: remove the whole VM runtime as `review` via the Trash

**Pros**:
- Frees tens of gigabytes with one line.

**Cons**:
- The VM image holds volumes with data; after the Trash is emptied, they are lost.

**Rejected because**: the decision "remove the runtime" requires knowing what is in the volumes, so a person makes it manually.

## Consequences

**Positive**:
- Predictable behavior: the class determines default selection and `--yes`, the method determines how removal happens.
- Background mode and the GUI cannot delete anything by accident; a VM cannot be removed as a directory at all.

**Negative / trade-offs**:
- Every rule needs manual classification, a marker and a method; new ecosystems are added in code.
- The rescan in `clean` costs time (seconds to tens of seconds). Once the amendment of 2026-10-24 lands, it is paid only by the terminal's `ddc clean` and by an `--items` list that needs the whole tree.
- Projects without a lock file and with unknown activity are never selected automatically; they have to be picked by hand.

**What changes in code / process**:
- The rule type in `ddc-core`: risk class, removal method, marker, context confirmation, native command, owner process, list of lock files.
- Executor: re-check, tombstone, `in-use` check, log.

## Amendments

- **2026-10-18, proposed.** [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]], §7 amends §5. An explicit item list, confirmed by a person in the app's confirmation sheet or named at a terminal with `ddc clean --items`, counts as interactive confirmation for each listed item, `review` and `native-only` included.
  - **What it changes.** Until now no non-interactive call could remove a `review` item, because `--yes` refuses that class. With `--items`, a script without a TTY can move named `review` items to the Trash, and, with `permanent: true` on an item, delete them permanently.
  - **What bounds it.** Consent is per item: a global `--permanent` with `--items` is refused, and each item's class and action must match the fresh scan, or it is skipped as changed since the scan. Everything else in §5 still applies to every item: the rescan (since 2026-10-24, the walk of that date's amendment), the re-checks and the refusals. In the app, the confirmation sheet's Clean action is the only caller of `clean`, and a test enforces it.
  - §8 is unchanged: the background path still never calls the executor.
  - The amendment takes effect when that ADR is accepted.
- **2026-10-20, decided by the owner.** Story "User data: pick your own files for the Trash" (epic DDC-002) amends §1, §2 and §5. The user's own files may enter a plan when the user picks them one by one in the app's Review window, or names them in an `--items` list, and only to the Trash.
  - **What it changes.**
    - **§1.** The `user-data` row's "never enters the plan" becomes: never by default, and an entry of a user-data folder can be picked by hand, to the Trash only.
    - **Entries.** An entry is a direct child (for backups, one backup folder) of a finding of `user.downloads`, `user.pictures`, `user.movies`, `user.music` or `user.ios-backups`. `user.imazing-backups` stays report only until its layout is checked.
    - **The `none` method** still covers the folders themselves, and every other `none` finding: messenger containers, iMazing backups, `user.kept` and the Trash. The cloud folders are a skip, not a finding: still not traversed. The executor relaxes `none` only for an entry of those five findings.
  - **What can be picked.** An entry the fresh scan lists, unlocked: a regular file or a directory, never a symlink, another kind or a dataless entry.
  - **Locked entries** are listed and never picked:
    - entries managed by an app (library packages and fixed paths, from a maintained list);
    - entries holding a `.ddc-keep`, or excluded, or holding an exclude (lock "kept");
    - protected folders (§6);
    - entries holding a mount point;
    - entries added or modified within 10 minutes of the scan's start (either side; the dry run measures against its own start; with neither time readable, locked), partial downloads, and backups that did not finish (a missing `Status.plist` counts as not finished).
  - **The marker and excludes (§2), by folder kind.**
    - A `.ddc-keep` directly in a folder, or an exclude of the folder or above it, still makes the whole folder `user.kept`.
    - In the four folders the scan claims whole (Pictures, Movies, Music, iPhone and iPad backups), an exclude anywhere inside still makes the whole folder `user.kept`. A `.ddc-keep` below the folder's top now locks the entry that holds it, and no longer turns the whole folder into `user.kept`. This is the one change to §2.
    - Downloads is a region and is unchanged: a direct child folder with a marker at its top, or an excluded one, is its own `user.kept` finding; a `user.kept` finding deeper inside an entry locks that entry. An excluded file, which the scan's exclude does not claim, is listed locked "kept".
    - For an entry the lock is checked before §5's marker walk, so a marker in or at an entry, or an exclude at or in a Downloads entry, is `entry-locked` (kept), in the clean and in the dry run. A Downloads folder that became a `user.kept` finding this way is `entry-locked` too, not `not-an-entry`. In the four folders the scan claims whole, an exclude anywhere inside keeps the whole folder, so its entries are skipped as `kept`.
  - **Re-checks (§5).** For these items, §5's rule-marker checks, written for directories found by rules, are replaced by:
    - the parent is the folder, with the identity of the fresh scan's finding, and the item's rule is that folder's rule;
    - the entry, opened through the parent's file descriptor, is a regular file or a directory with the listed `(dev, ino)`;
    - the fresh scan still holds it as an unlocked entry, checked against the folder's full list of entries, not the report's top 100;
    - every other §5 check still runs: inside the scan roots, no symlink above the parent, protected paths, in use, `.ddc-keep`. Only four are replaced: the rule-marker match, the `.git` or project-root check, the `incomplete` check and the directory-only check.

    The entry runs before any finding inside it, and that finding is skipped as `inside-another-item` only when the entry went to the Trash; if the entry was skipped or failed, it runs. The new skip reasons come with a raised clean-report schema version. The in-use check does not see open files: an open file can reach the Trash, and the Trash keeps it. (Note, 2026-10-20, 0.2.1: the check now sees open files for `delete` and `native` items; a move to the Trash still never counts them. See the amendment for story "Release 0.2.1".)
  - **What bounds it.**
    - The action is `trash`: an `action: delete` item is skipped as `action-not-allowed`, and `permanent: true` refuses the whole list when it is read (exit 2).
    - `--permanent` with `--items` is refused by the argument parser.
    - `--yes` and `--class` never select user data, and nothing is picked by default.
    - §8 is unchanged: the background path never calls the executor.
  - **The cost accepted.** One `--items` run can send many large entries, whole device backups included, to the Trash. The Trash is the only undo, and the space comes back when the user empties it. The mitigation is in the app: no default selection, no Select all for user data, and a "Your own files" block that comes first in the confirmation sheet.
- **2026-10-20, decided by the owner (0.2.1).** Story "Release 0.2.1: the Review window in the Dock, and skip what a running process holds open" (epic DDC-002) amends §5's in-use check. Cause: a clean from the 0.2.0 app waited the full 600 s `NATIVE_TIMEOUT` on `uv cache clean`, because long-running `uv run` processes held uv's cache lock through an open file in `~/.cache/uv`, which the check did not see.
  - **Open files count.** For an item removed by deletion (`delete`) or by a tool's own command with a fallback (`native`), a process of the current user that has a file open inside the finding (`proc_pidinfo(PROC_PIDLISTFDS)`, `proc_pidfdinfo(PROC_PIDFDVNODEPATHINFO)`, vnode files only) puts the item under `in-use`, as its executable, arguments and working directory already do. A hit is confirmed by reading that one process's files again after about 250 ms, so a passing reader such as a Spotlight worker does not block the item. Before each `native` command the open files are read again for that item, since a long clean reaches it minutes after it started. Every in-use reason, the path-based ones included, names the process, or its app when it lives in an app bundle.
  - **A move to the Trash never counts open files**, as the user-data amendment says: the Trash keeps an open file working. This covers `trash` items, `review` items to the Trash and picked user-data entries.
  - **`native-only` items are exempt from the whole in-use check**, the path-based part included. Their runtime's daemon lives inside the finding by design (Docker Desktop's `com.docker.backend` runs with its working directory in `~/Library/Containers/com.docker.docker`), its prune commands are what knows what is in use, and the executor never removes their files itself (§1). Until 0.2.1 the working-directory check skipped Docker's prune whenever Docker ran.
  - **The scan's default selection does not consult processes**; only the clean does, at execution and in its dry run.
  - **Tool commands** may carry environment variables; `cache.uv` sets `UV_LOCK_TIMEOUT=10`, so a lock taken after the check fails in seconds (uv 0.9.16 and later; older uv ignores it and `NATIVE_TIMEOUT` stays the bound). `uv cache clean --force` is never used.
  - The clean report's schema version is unchanged: no new skip reason.
- **2026-10-24, decided by the owner on 2026-10-21.** Story "Faster clean: re-check only what the selection needs" (epic DDC-002) amends §5's rescan for an explicit item list. Cause: a clean of two small caches from the app waited about 29 s before its first action. The wait is attributed to the full rescan, because a full scan of the same home took 37 s; the story times each step before the change.
  - **Rule scope.** Every rule declares what its verdict needs the walk to have visited, in a field matched exhaustively, so no rule exists without one. A path a rule stats or reads on its own does not count, because the pruned walk makes the same call: a sibling lock file, the `CMAKE_HOME_DIRECTORY` of a `CMakeCache.txt`, `~/.nvm/alias/default`. The scopes:
    - `Item`: the item, its parent and its ancestors (almost every rule);
    - `Siblings`: also the other children of its parent (`ide.jetbrains-old-data`, whose newest sibling is kept);
    - `Tree`: files anywhere in the scan roots. These are `toolchain.nvm`, `toolchain.pyenv` and `toolchain.rustup`: their versions are protected by pin files in any project, and with one claimed version or none, every version is kept (none is a finding).
  - **Which walk.**
    - **The full walk, as before:** a list holding a `Tree` item. Two more cases take it for reasons other than safety (three since 2026-10-25):
      - a run with no readable report of the current schema, because the CLI edits that report after the clean ([[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]], §6 item 1);
      - a list with an item whose ancestor cannot be read (found in the walk's errors). This is a conservative choice: the full walk cannot read that ancestor either, and the item ends `changed-since-scan`.
      - added 2026-10-25 at the story's plan gate: a report on disk whose `home` or `scan_roots` differ from the clean's, or one that cannot be read. `ddc scan ~/Code` writes a report of `~/Code` alone, and editing it would leave the app showing that report after a clean of the whole home.
    - **The pruned walk, for any other list:** the same rule visitor, classification and plan as the full walk, over the listed items, their ancestors up to the scan root, and the siblings of a `Siblings` item with their subtrees. A picked user-data entry's folder is listed with times, so the entry is still checked against the folder's full list of entries (amendment of 2026-10-20).
    - A claimed ancestor makes the walker measure that ancestor's whole subtree, because the walker's hooks are off inside a claim. For a picked entry of Pictures, Movies, Music or the device backups that ancestor is its folder, claimed whole by design. Downloads is a region whose hooks stay on, so its other entries are pruned. For any other item the claim refuses it anyway, and the story's plan gate decides whether the walk stops there.

      Decided 2026-10-25 at that plan gate: the walk stops there. The walker asks the pruning check before it opens any directory, inside a claim too. A claimed ancestor's other subtrees are therefore not read, and neither are the unpicked entries of Pictures, Movies, Music or the device backups. Their folder is still listed with times, so a picked entry is checked against the folder's full list of entries. Under a claimed ancestor the plan decides from the claim existing at the ancestor and the item not being its own finding, never from the claim's size, so the item is refused as the full walk refuses it. One reason can differ: when a claim appeared above the item after the scan and a keep marker sits in another of its subtrees, the full walk refuses the item `kept` (the marker turned the claim into `user.kept`), the pruned walk `changed-since-scan`. In the user's own folders a marker locks only the entry holding it (amendment of 2026-10-20), so a picked entry is decided the same either way. The pruned walk's partial sizes never reach `last-scan.json`: in the pruned case the CLI edits the report it read before the clean.
    - A listed item outside the current scan roots, or with a rule the catalog lacks, is refused as today and needs no walk.
    - There is one classification path: the pruned walk only skips directories.
  - **Why it sees what the full walk sees:**
    - a new claim above an item is seen on the way down through its ancestors;
    - name-with-sibling matchers stat the parent, and content matchers read inside the item;
    - excludes are path prefixes;
    - a keep marker inside an item is found by the marker walk, and one at or above it by the guard at the moment of acting;
    - a lock file or `.git` that changes the class or the project root lies in an ancestor;
    - a hard link with a link outside the item counts as shared in both walks: the full walk sees the other link, the pruned walk sees fewer links than `nlink`;
    - "inside another item" compares the list with itself.

    What a pruned walk cannot see, a file anywhere else, is exactly what makes a rule `Tree`.
  - **Unchanged:**
    - the matching of a listed item against the plan: path, `(dev, ino)`, rule, class, action;
    - the per-item re-checks before each action, and the refusals;
    - the terminal's `ddc clean`, `--yes`, and §8.
  - **The cost accepted.** A rule's scope is a claim made in code: a future rule that reads outside its item but is declared `Item` would be judged on a partial view. Two things guard against it: the exhaustive field, and equivalence tests in which the pruned and the full re-check agree for every rule of the catalog, for each kind of change after the scan. A test checks the scope sets against `toolchain::dropped`, the one place that consumes whole-walk state, and fails when they differ. An equivalence test fails if `ide.jetbrains-old-data` gains a second anchor, which would make `Siblings` insufficient.
  - **Gained besides.** A terminal `clean --items` of caches no longer reads Downloads, Pictures or the device backups, so it no longer makes macOS ask for access to them.
  - The amendment takes effect when the story's code lands.
- **2026-10-28, decided by the owner.** Story "Settings: folders, types, notifications" (epic DDC-002) amends §2 item 1. A folder can be excluded in one of two ways:
  - **Count, never clean.** This is today's exclude: the folder is read and sized, and it becomes `user.kept`.
  - **Skip entirely.** The folder is listed in `settings.json`'s new `skips`, or named with `--skip`, which only adds to that list. A skip must be a directory. The walk stops before reading it, so it is never opened. The report's new optional `not_scanned` lists the skips the walk stopped at, and the Review window shows them as not scanned.
  - **Skips also protect.** A skip is added to the excludes as well, so a finding that holds a skipped folder is kept whole. `clean --items` refuses an item at, under or above a skip as `kept`, in the dry run and in the clean alike.
  - **Flags cannot drop a skip.** An `--exclude` flag never removes a skip, and the settings file is the only place to delete one. For that reason the `--exclude` confirmation of the menu bar ADR §6 item 6 does not apply to skips.
  - **Inside a claimed subtree** the walker's hooks are off, so a skip there is still read and sized. It is protected as an exclude is, the finding is kept instead, and the skip is not listed in `not_scanned`.
  - The amendment takes effect when the story's code lands.
- **2026-10-28, decided by the owner.** Story "Whole disk: free space with the owning tool" (epic DDC-002) amends §5, §6 and §7. Outside the home folder ddc still removes nothing by path. Two areas of the whole-disk report have a tool that frees them safely, and a person can now run that tool after a confirmation.
  - **The area action, a new kind of native action.** It is not a rule of the catalog, it has no finding, and it has no path inside the scan roots. There are two, named by the disk report's `action` ids:
    - `simctl-runtime-delete`: `simctl runtime delete <identifier>`, one simulator runtime, where `identifier` is the asset id the disk report lists for that runtime;
    - `brew-cleanup`: `brew cleanup` with no arguments, and `HOMEBREW_NO_AUTOREMOVE=1`, `HOMEBREW_NO_ENV_HINTS=1` and `NO_COLOR=1` in its environment. Without the first, current Homebrew's `brew cleanup` also uninstalls every formula that was installed only as a dependency and that nothing installed still needs (`Cleanup.autoremove`, on by default). ddc's action removes old versions and downloads, never a formula. The dry run, `brew cleanup -n`, runs with the same environment, so the preview and the run follow the same rules.

    The tool's own command is the only way it frees space: ddc removes no file of an area itself, and there is no fallback.
  - **The guard is a location probe, in place of §5's re-checks**, which are written for a finding. It runs immediately before the command, and again in the dry run. Folders are compared by `(dev, ino)`, not as strings, except where the text below says otherwise (a `parentMountPath` prefix, a cache folder that does not exist yet): the disk report spells an area through `/System/Volumes/Data` (`/System/Volumes/Data/opt/homebrew`), the tools answer through the firmlink (`/opt/homebrew`), and `canonicalize` does not unify the two.
    - **A runtime.** The listed identifier is matched byte for byte against the `identifier` of exactly one runtime in a fresh `simctl runtime list -j`, before any `simctl` runs with it, and is then passed as the first argument after `delete`, followed only by `--dry-run` in the dry run. `all`, a build or a version, `--notUsedSinceDays`, `--unusable` and `--outdated`, which `simctl runtime delete` also accepts, therefore never reach it. That runtime:
      - has a `parentMountPath` that starts with `/System/Library/AssetsV2/`;
      - is not `deletable: false` and not in the state `Deleting`;
      - has a `runtimeIdentifier`, so its devices can be joined to it, and no device on it is booted. `simctl runtime delete` would shut a booted device down itself; a person's running simulator is not ours to stop.

      `simctl` is found as `ddc disk` finds it (story "Whole disk: what lies outside the home folder"): never `xcrun`, never the shim under `xcode-select -p`, only CoreSimulator's binary, after `xcodebuild -checkFirstLaunchStatus` exits 0. `--keep-asset` is never passed: it keeps the runtime's asset and would free nothing. The runtime's folder is its `<hash>.asset` directory under AssetsV2, the one that holds its `path`.
    - **Homebrew.** `brew` is `<prefix>/bin/brew` of the Homebrew area, `/opt/homebrew/bin/brew`, not the first one on the tool path. `brew --prefix` is, by `(dev, ino)`, the Homebrew area's folder; `brew --cellar` lies inside it; `brew --cache` is, by `(dev, ino)`, the `cache.homebrew` rule's path, `~/Library/Caches/Homebrew`. When that folder does not exist, which is the case right after a clean removed the `cache.homebrew` finding, the two are compared as paths instead: both lie under the home folder, with no firmlink between them, and `brew cleanup` recreates the folder itself. The last check is about the lock and the report: the cache is the one place `brew cleanup` writes inside the scan roots, and ddc re-measures that finding afterwards. An Intel `/usr/local` Homebrew, or a `HOMEBREW_CACHE` set elsewhere, refuses the action.
    - A refused action runs nothing and says why.
  - **Consent (§5).** An area action runs only from a person's explicit confirmation of that action:
    - the app passes the confirmed list with `ddc disk free --actions FILE|-`, as `clean --items` passes items (menu bar ADR §6 item 11, §7). That lets a script without a TTY run `brew cleanup` or delete a runtime; it is the cost of a machine-readable consent channel, as for `--items`;
    - at a terminal without `--actions`, `ddc disk free` lists the available actions with their dry-run estimate and asks for each one, default No, before it takes the lock;
    - without a terminal and without `--actions` it refuses (exit 2). There is no `--yes`: §5 keeps `--yes` for default-selected `safe` items.

    §8 is unchanged: the background path and the scheduler never call `disk free`.
  - **The lock (§5).** `ddc disk free` takes the exclusive lock, for both actions, because `brew cleanup` writes inside `~/Library/Caches/Homebrew`, a scan root, and the run edits `last-scan.json`. A held lock exits 75. Its dry run takes the shared lock, as `clean --items --dry-run` does.
  - **What it measures.** The freed bytes are measured, never assumed:
    - a runtime: its `<hash>.asset` folder's allocated bytes before, and after the runtime has left `simctl runtime list -j`. CoreSimulator's daemon deletes it after `simctl` returns, so the command waits for the runtime to leave the list. The command and the wait share one `NATIVE_TIMEOUT`; past it, the runtime is reported as still deleting. A runtime that left the list while its folder did not shrink is reported "removed, not freed" with the bytes, so ddc never reports space it did not free. A stop during the wait ends the wait, writes the journal line with the result `still-deleting`, makes the report edits of menu bar ADR §6 item 11, and then ends ddc by its signal: it neither stops nor undoes the deletion, which the daemon performs;
    - Homebrew: the Cellar and the cache before and after. Afterwards the `cache.homebrew` finding in `last-scan.json` is measured again, as a clean does for an item a tool cleaned, when that report has the current schema and the home's roots (as `clean --items` requires).
  - **Running the tools (§6).** Both run through the commands port, without a shell, stdin closed, with `NATIVE_TIMEOUT`, in their own process group, so a stop reaches them as it reaches a rule's command. A non-zero exit, a timeout or a signal is `failed`, with the tail of the tool's output and the exact command for the person to run in Terminal. It is worded as needing an administrator only when the output names a permission error (`Permission denied`, `Operation not permitted`); otherwise it says the tool failed. ddc never asks for a password itself.
  - **The journal (§7).** Every run of an area action writes one `actions.jsonl` line of a new kind: `kind: "area-action"`, the action id, the area, the runtime's identifier where there is one, the exact command, the allocated bytes before and after, the result, and the tail of the tool's output. It has no `path`, `(dev, ino)`, `rule` or `class`. A dry run writes nothing.
    - The journal gets its own `JOURNAL_SCHEMA_VERSION`, as the clean report and the progress events have theirs. Until now its lines carried the scan report's `SCHEMA_VERSION`, and bumping that would make every `last-scan.json` unreadable.
    - Lines written so far carry `schema_version: 1`, the scan report's current value, so the journal's own version starts at 2. Every line written from now on carries `kind`: `item` for a clean's line, whose fields are otherwise unchanged, or `area-action`. A line of version 1 has no `kind` and is an item line.
  - **The cost accepted.**
    - `brew cleanup` removes old versions of installed formulae that are neither linked nor pinned (Homebrew skips those), downloads of versions no longer installed and downloads older than `HOMEBREW_CLEANUP_MAX_AGE_DAYS` (120), Homebrew's old logs in `~/Library/Logs/Homebrew`, stale lock files, and broken symlinks in its prefix. It does not remove old cask versions from the Caskroom. A person who kept an old, unlinked, unpinned keg on purpose loses it. The confirmation names the Cellar and the cache, lists what `brew cleanup -n` prints, and says that `HOMEBREW_*` variables set in a shell profile do not reach a run started by the app, while Homebrew's own `brew.env` files do.
    - A deleted runtime of the installed Xcode's SDK is downloaded again, several GB, at the next build for that simulator; the confirmation says so.
  - The amendment takes effect when the story's code lands.
- **2026-10-29, accepted at the story's work-start gate.** Story "Whole disk: Conda environments and its package cache, with conda" (epic DDC-002) amends §5 and §6. Conda environments and package caches become findings in the home folder, cleaned by conda, and an `/opt` install gets two area actions.
  - **Which conda (§6).** Conda is never the first `conda` on the tool path. The known installs are `~/miniconda3`, `~/anaconda3`, `~/miniforge3`, then `/opt/miniconda3`, `/opt/anaconda3`, `/opt/miniforge3`, in that order; each is run as `<prefix>/bin/conda` through the commands port. Each is asked `conda config --show root_prefix envs_dirs pkgs_dirs allow_softlinks always_softlink --json`, which prints conda's computed settings; conda itself writes nothing for it; it is cheap and deterministic, and the Anaconda Terms of Service plugin does not run for it. `conda info --json` and `conda env list --json` are not used to ask: both list environments, which rewrites `~/.conda/environments.txt` when an entry there is no longer an environment. A probe that fails, times out or prints JSON ddc cannot read claims nothing. The probe runs with ddc's own environment, so a `CONDARC` or `CONDA_ENVS_DIRS` set where ddc started shapes the answer: the tool's own view, accepted. Every candidate install that exists is asked, at most once per clean or per area-action run; the preflight asks no install and predicts the command from a candidate `bin/conda` existing.
  - **"The tool reports that path as its own" (§6), for conda.** Folders are compared by `(dev, ino)`; listed folders that do not exist are skipped.
    - An environment is conda's when its parent folder is one of an install's `envs_dirs`, it holds `conda-meta/history`, and it is not that install's `root_prefix` (the base environment is never removed).
    - A package cache is conda's when it is one of an install's `pkgs_dirs`.
    - The first install, in the order above, that claims the finding runs the command, and the journal names its `bin/conda`.
    - A finding path that is not UTF-8 or contains a comma is claimed by no install.
    - conda receives the firmlinked path (`/opt/...`, never `/System/Volumes/Data/opt/...`), so its own comparison with its `root_prefix` stays a second refusal of the base.
  - **The commands.**
    - An environment: `conda env remove -p <path> --yes`. Never `-n`/`--name`, never the base. It deletes the environment and rewrites `~/.conda/environments.txt` to drop it; releases before 24.9 also remove the parent `envs` folder, and its empty parents, when it is left empty. Both are admitted, outside the finding: the file is conda's own list of environments, rewritten, not removed, as conda rewrites it whenever it lists environments and finds an entry gone; the folders removed are empty. It runs conda's unlink transaction first, so a package's pre-unlink script may run, as in Terminal. On an Anaconda Distribution install the Terms of Service plugin runs before it; with stdin closed and the Terms not accepted, it fails with the plugin's message and removes nothing.
    - A package cache: `conda clean --index-cache --packages --tarballs --logfiles --yes` with `CONDA_PKGS_DIRS=<that folder>` in its environment. Never `--all`, `--tempfiles`/`-c` or `--force-pkgs-dirs`: the temp-file part walks the whole install of the conda that runs, which the variable does not bound. Without the variable, conda would clean every package folder it can write, so a home install's `pkgs` and `~/.conda/pkgs` would both go. The variable does not bound it alone: `pkgs_dirs` is a sequence, and conda merges a sequence from every source, so a `.condarc` listing two package folders would still have both cleaned. So right before each clean and its dry run the claiming install is asked `conda config --show pkgs_dirs --json` with the same `CONDA_PKGS_DIRS`; entries that do not exist are dropped, as conda's clean drops them, and the clean runs only when what remains is exactly the finding's folder, by `(dev, ino)`; a second existing folder, writable or not, skips the cache as `tool-not-ready`. With that verified and the four flags, the command acts on `<folder>`, `<folder>/cache` and `<folder>/.logs` only, plus, on conda 26.9 and later, the same rewrite of `environments.txt` as above (it lists environments to find softlinked packages), in the run and in its dry run. It removes packages no environment uses, tarballs, the index cache and conda's logs: less may be freed than the folder's size, and the confirmation says so. Before 26.9, "no environment uses" is decided by the hard-link count alone, so a package softlinked into an environment counts as unused. Therefore when any install whose `pkgs_dirs` lists the cache answers `allow_softlinks` or `always_softlink` true, the cache is skipped as `tool-not-ready` and no command runs (`~/.conda/pkgs` is in every install's list). Both are off by default. On 26.9 and later conda checks softlinks only for environments in `environments.txt`; the skip stays. Accepted: a package softlinked while a setting was on that is off now counts as unused on conda before 26.9, as it does for conda in Terminal. Afterwards the finding is measured again, as for every tool-cleaned finding.
  - **Fallbacks.** An environment no install claims goes to the Trash, as the rule's fallback. A package cache no install claims is skipped as `tool-not-ready`, naming conda: it is never deleted or moved to the Trash without conda, because with `allow_softlinks` or `always_softlink` environments' files are symlinks into the cache, and removing it would break them. This is the first native rule whose fallback is a skip.
  - **In use (§5).** A `conda` environment finding is a native item, so the in-use check of §5 applies to it in full, including a process whose `CONDA_PREFIX` is the environment.
  - **The `/opt` area actions.** Two actions join the area actions of the 2026-10-28 amendment, with its consent, lock, journal line, measuring and running rules unchanged:
    - `conda-env-remove`: `conda env remove -p <identifier> --yes`, where `identifier` is an environment the disk report lists under an `/opt` install;
    - `conda-clean`: the cache command above with `CONDA_PKGS_DIRS=<prefix>/pkgs`, where `identifier` is the install's prefix, and the same softlink skip. Its dry run is the same four flags with `--dry-run --json` and the same environment.

    An environment's files are hard links into `<prefix>/pkgs`, so removing it frees only its bytes not also in the cache. Its dry run reports, and its run measures before and after, only the environment's own bytes: the files whose every link lies inside it, which the walker already counts for a region. A link into a package folder or another environment is never reported as freed; the confirmation says the rest is freed by cleaning the package cache. This keeps the 2026-10-28 amendment's rule that ddc never reports space it did not free.

    The disk report lists these environments from the file system (`<prefix>/envs/*` holding `conda-meta/history`); `ddc disk` never runs conda. The location probe, immediately before the command and again in the dry run, refuses with its own reason: a prefix that is not one of the three `/opt` prefixes by `(dev, ino)`; an install whose `root_prefix` is not that prefix; an environment whose parent is not in `envs_dirs`, that lacks `conda-meta/history`, that is the base, or that lies in the home folder (that one belongs to the catalog rule, with §5's re-checks); a `<prefix>/pkgs` not in `pkgs_dirs`; and an environment a running process uses, by §5's in-use check, which the probe runs because an area action has no §5 re-check of its own. An environment or a `<prefix>/pkgs` the user cannot write is refused by the probe before conda starts, with the administrator wording: conda's own refusal is not worded as a permission error, and in two cases it exits 0 having done nothing (an environment with no package records, a package folder it cannot write). An environment still present after conda exits 0 is "removed, not freed", never freed, and stays in the disk report; a catalog environment finding in the same state is failed. ddc never asks for a password.
  - **Schemas.** The two action ids join the shared action-id enum and the probe's refusals join the refusal enum, so the disk report's and the free report's schema versions are raised. The scan and clean reports are unchanged.
  - The amendment takes effect when the story's code lands.
- **2026-10-29, accepted at the story's work-start gate.** Story "Whole disk: Homebrew packages, their sizes, and brew uninstall" (epic DDC-002) amends §6 and the 2026-10-28 amendment. The 2026-10-28 amendment lets the owning tool free its own area: old versions and downloads. This one lets brew uninstall a formula a person picked, and the formulae that removal leaves orphaned.
  - **Formulae only.** Every uninstall is `brew uninstall --formula`, so a formula removed since the listing can never fall through to a cask of the same name (`brew uninstall docker` would remove Docker Desktop). Casks, `--zap`, `brew autoremove` and `sudo` are never run. Every command names a formula by its `full_name`.
  - **The commands.** Two actions join the area actions of the 2026-10-28 amendment, with its consent, lock, `area-action` journal line and running rules unchanged; what differs is below.
    - `brew-uninstall`: `brew uninstall --formula <full_name>` for a formula with one keg, and `brew uninstall --formula --force <full_name>` for one with several. `--force` removes every keg and skips brew's own pinned check and per-keg lock; brew still refuses dependents before touching a keg, and a file it cannot remove still stops it with an error.
    - `brew-uninstall-orphans`: one `brew uninstall --formula --force <a> <b>…` with the formulae the first removal left orphaned, after their own confirmation. brew removes them rack by rack and stops at the first it cannot remove; the ones before it are gone, which the confirmation says.
  - **The environment.** The 2026-10-28 amendment lists `brew cleanup`'s three variables exactly. `brew info`, `brew services list` and `brew uninstall` run with the same three (`HOMEBREW_NO_AUTOREMOVE=1`, so an uninstall never removes other formulae itself, `HOMEBREW_NO_ENV_HINTS=1`, `NO_COLOR=1`) plus `HOMEBREW_NO_AUTO_UPDATE=1`, so none of them refreshes Homebrew's API files first when they are only stale (a missing API folder is still downloaded). The uninstall path's location probe (`brew --prefix`, `--cellar`, `--cache`) runs with the same four; `brew-cleanup`'s probe and command keep exactly the three.
  - **The re-read, in place of a dry run.** brew has no dry run for `uninstall`. Immediately before each uninstall, after the 2026-10-28 location probe, ddc reads `brew info --json=v2 --installed` again, and `brew services list --json` where brew has it, and recomputes dependents and orphans from it. It refuses, naming why, when a formula is no longer installed, is now pinned, has kegs other than exactly the confirmed ones, has gained a dependent (named), is no longer an orphan, or has a service job: a `status` other than `none`, a `user` of `root`, or a plist registered under `~/Library/LaunchAgents` but not loaded. This is the pinned check `--force` skips, and the only check that notices a formula already gone (`--force` exits 0 having removed nothing). Dependents are matched on the short name, and a keg with no recorded runtime dependencies is read through its declared `dependencies` and `recommended_dependencies`, because brew's own refusal does not see that keg's needs. `ddc disk free --dry-run` runs the re-read without the command.
  - **The orphans** are ddc's computation, never brew's `autoremove`: the removed formula's runtime dependencies, recursively, not installed on request, that no remaining formula and no installed cask's `depends_on formula:` needs. A pinned one is kept. They are removed only by the second action, from its own confirmation listing each by name and every keg. That action carries names, not kegs: its re-read checks that each is still installed, unpinned, without a dependent or a service job, and still an orphan, its `names` taken as the set: not installed on request and needed by no formula or cask outside the set; an orphan upgraded since its confirmation loses its new keg as well, accepted.
  - **What it measures.** The freed bytes are the Cellar's: the confirmed formulae's racks (`<cellar>/<name>`, by the re-read's `name`), measured before and after; the rest of the Cellar is not walked. The cache is left out: an uninstall writes nothing there, except Homebrew's API files when their folder is missing, which would make the cache grow and the freed bytes shrink.
  - **Where it writes outside the Cellar.** Besides that API download into `~/Library/Caches/Homebrew/api`, current brew records an uninstalled tap formula as no longer trusted in `trust.json` under `~/.homebrew/` (`${XDG_CONFIG_HOME}/homebrew/` when that variable is set), and only when the formula had its own trust entry, so most uninstalls write nothing there. Both are inside the scan roots and under the exclusive lock; both are Homebrew's own files, admitted.
  - **The budget.** The probe, the re-read and the uninstall share one `NATIVE_TIMEOUT`, so the exclusive lock is held for at most one `NATIVE_TIMEOUT` per action. `brew uninstall` does not start when less than a floor of it remains; that result says nothing was removed.
  - **The cost accepted.** brew does not stop a running service job and leaves the formula's data in `var/` and `etc/`; the first is refused, the second is said in the confirmation. brew cannot know about users outside brew, such as a virtualenv linked against brew's openssl; the confirmation says so. A Stop or a timeout during a `--force` removal (several versions, or the orphans) can leave some kegs removed, and the keg it was removing half removed; the sheet lists again. A `failed` that stopped before `brew uninstall` started (the floor, an unreadable re-read) still marks the Homebrew row as measured before cleanup: conservative, not a bug. ddc's orphan set is wider than brew's `autoremove` in two ways: a keg whose record predates `installed_on_request` counts as not installed on request (brew's JSON prints `false` for it, where `autoremove` keeps it), and formulae built from source are in the set (`autoremove` keeps them). The second confirmation lists every one by name, so the person decides.
  - The amendment takes effect when the story's code lands.

## References

- [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]]
- [[adr/rust-core-library-and-cli-with-json-output|Rust core: library + CLI with JSON output]]
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]
- Cache Directory Tagging Specification (bford.info/cachedir)
- Rust `std::fs::remove_dir_all` (CVE-2022-21658)

---

*Last updated: 2026-10-29*

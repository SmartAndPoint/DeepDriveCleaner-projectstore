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
reviewed_at: 2026-10-17
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
| `user-data` | user data | photos, videos, backups, downloads, messengers, cloud folders | **never enters the plan** | — |

Removal methods:

| Method | What it does | Where it is used |
|---|---|---|
| `native` | the tool's native command | global caches that have one |
| `delete` | directory removal via a tombstone (item 5) | `safe`, `rebuildable` with effective class `rebuildable` |
| `trash` | move to the Trash (`NSFileManager.trashItem`, no AppleScript) | `review` by default, including `rebuildable` downgraded to `review` (item 3); `--permanent` replaces it with `delete` |
| `native-only` | **only** the native command, and only when the daemon is running; otherwise a report line with instructions. Only commands that delete neither containers nor volumes are allowed, and only with an explicit target: `docker --context desktop-linux builder prune -f`, `docker --context desktop-linux image prune -f` (no `-a`; if that context does not exist, instructions only), `podman image prune -f`. Space is freed inside the VM image (`Docker.raw`), and `statfs` may not change until the runtime shrinks the image; the plan line says so. `system prune` of any runtime is forbidden: it deletes stopped containers together with their data | VMs and stateful containers: Docker Desktop, podman, colima, lima, OrbStack, Rancher Desktop |
| `none` | report only | `user-data`, Trash |

For `native-only`, removing a directory or a file is forbidden: the executor itself checks the plan's action against the rule's method and refuses (`action-not-allowed`) if the plan asks for something the rule does not allow — `delete`/`trash` for `native-only`, any action for `user-data` and `none`. The plan is not treated as trusted input: a GUI or another client of the core can build it. The plan line for a native command names the command and what it deletes.

### 2. Rule priority

The walk goes top-down; for each directory, the decision is made in order, and the first match wins:

1. exclusions: `--exclude PATH`, a `.ddc-keep` file in the directory or in any of its ancestors inside the scan root → a finding with class `user-data` and the label "protected by user". The presence of `<dir>/.ddc-keep` is checked before each directory is read (`faccessat`), so a marker inside a known path (`~/.cache/uv/.ddc-keep`) also takes effect. If `.ddc-keep` is found anywhere inside an already claimed subtree (`node_modules/foo/.ddc-keep`), the whole finding becomes `user-data` "protected by user": partial removal is not supported; cloud regions (`~/Library/CloudStorage`, `~/Library/Mobile Documents`) and dataless directories are not walked;
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
- the finding's path is not in use by a running process (item 5).

### 4. Project activity

The project root is the nearest ancestor of the finding that has `.git` (a directory, **or a file** for a worktree), otherwise the manifest's directory. Activity is the maximum mtime among: `<gitdir>/index` and `<gitdir>/HEAD` (for a worktree, `gitdir` is read from the `.git` file), and the project's manifests and lock files. If there is no signal at all, activity is unknown and the item is not selected.

### 5. Plan, confirmation and execution

- `ddc scan` only reads.
- `ddc clean` **always rescans** and builds the plan from the fresh scan, shows it (what, how much is allocated, how much it frees — the total is `~` (an estimate; `≥` on an individual finding is shown by `ddc scan`, where the private size is measured and noticeably smaller than allocated); for `native-only` the space is freed inside the VM image and is not included in the total — and how it is removed) and waits for confirmation. `--dry-run` prints the plan and exits.
- `--class` is repeatable and works as a filter (`--class safe --class rebuildable`). `--yes`, non-interactively, removes only items **selected by default**, and only from the classes listed in `--class` (default `safe`). With `--yes`, class `review` in `--class` is not accepted (error). `review` and `native-only` always require interactive confirmation.
- The scan root itself is never removed: `ddc clean ~/.cache/uv` refuses; you need to scan the parent.
- A scan takes a shared file lock, a cleanup takes an exclusive one (`flock` on `<data-dir>/lock`). A background scan skips the run when the lock is held.
- The data directory (`last-scan.json`, the lock) and the log directory are set by `--data-dir` / `DDC_DATA_DIR`; by default `~/Library/Application Support/DeepDriveCleaner` and `~/Library/Logs/DeepDriveCleaner`. Tests always pass a temporary directory.

Before each action, the executor re-checks the finding (protection against TOCTOU and against "same path, different directory"):

- the path exists and is a directory, not a symlink;
- the `(dev, ino)` of the finding and of its parent match the values recorded during the scan; otherwise skip with reason `changed-since-scan`;
- the rule's marker still matches;
- the path lies inside the scan root, is not a protected folder (item 6), has no `.git` at its top level and is not a project root;
- the finding was read completely: if it contained unreadable directories or another filesystem was mounted inside it, removal is rejected (`incomplete`);
- after the rename to a tombstone and before the Trash or the fallback, `(dev, ino)` is checked once more through the same parent descriptor (a tool probe can run between the check and the action);
- there is no `.ddc-keep` in the finding (at any depth) or in its ancestors inside the scan root;
- the path is not in use by a process: no process of the current user has, inside the finding, its executable path (the first string of the `sysctl(KERN_PROCARGS2)` buffer — the path passed to `execve`, before symlink resolution: `.venv/bin/python` is a symlink to an interpreter outside the venv), a command-line argument, its working directory (`proc_pidinfo(PROC_PIDVNODEPATHINFO)`) or a `VIRTUAL_ENV`/`CONDA_PREFIX` environment variable. A relative executable path and relative arguments are first joined with the process's working directory (`./.venv/bin/python app.py`). If a path is relative and the working directory cannot be determined, all plan findings whose name matches the first component of the relative path are blocked (`./.venv/bin/python` → all `.venv`), with reason `in-use-unknown-cwd`. Otherwise skip with reason `in-use`;
- for rules with an owner process (for example, `Spotify` for its cache), that process is not running.

Removal (`delete`): the directory is first renamed to `<name>.ddc-deleting-<time>` with `renameat` relative to the parent descriptor already verified by `(dev, ino)` (a single operation), then removed without following symlinks. On `EACCES`, the executor adds `u+w` to directories inside the finding and retries. An interrupted removal leaves a tombstone, which the next scan finds as `safe`.

Before cleanup, local Time Machine snapshots are checked (`tmutil listlocalsnapshots /`). If there are any, the user gets a warning that the space will be freed only after the system deletes them.

The summary is shown separately: deleted, moved to the Trash, actually freed (by `statfs` before and after, with the caveat that APFS frees space asynchronously).

### 6. Protected folders

**The folders themselves** are protected, not their contents. The `delete`/`trash` executor never removes: `/`, `/Users`, `$HOME`, any scan root, `~/Documents`, `~/Desktop`, `~/Pictures`, `~/Movies`, `~/Music`, `~/Downloads`, `~/Projects`, `~/Library`, `~/Library/Caches`, `~/Library/Application Support`, `~/Library/Containers`, `~/Library/Group Containers`, `~/.config`, `~/.ssh`, `~/.gnupg`, `~/.local`, `~/.cache`, `/System`, `/Applications`, `/Library`, `/usr`, `/opt`, `/opt/homebrew`. Neither removed nor walked: `~/Library/Mobile Documents`, `~/Library/CloudStorage`. iPhone backups (`~/Library/Application Support/MobileSync/Backup`) are a `user-data` finding: the size is counted (Full Disk Access is required), rules inside are not applied. Native commands work inside their own regions and are not limited by folder protection; therefore the rule catalog admits only commands that act strictly within the finding (for example, `uv cache clean`, `go clean -modcache`), and only if the tool itself reports that path as its own: for caches, the path from the tool (`uv cache dir`, `go env GOMODCACHE`/`GOCACHE`) equals the finding's path; for toolchains, the root from the tool (`pyenv root`, `rustup show home`) equals the root that contains the finding; otherwise, the rule's fallback is used. The command runs with a 10-minute timeout. `brew cleanup`, which removes old formula versions in `/opt/homebrew`, is not used in v1: the Homebrew cache is removed as a directory.

### 7. Log

Every action is a JSON line in `<log-dir>/actions.jsonl` with `schema_version`, time, path, `(dev, ino)`, rule, class, method, allocated size, private size (where it was measured), result, skip reason and the tail of the native command's output.

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

**Rejected because**: the user chose the "scan only" mode. It can come back later as an option.

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
- The rescan in `clean` costs time (seconds to tens of seconds).
- Projects without a lock file and with unknown activity are never selected automatically; they have to be picked by hand.

**What changes in code / process**:
- The rule type in `ddc-core`: risk class, removal method, marker, context confirmation, native command, owner process, list of lock files.
- Executor: re-check, tombstone, `in-use` check, log.

## Amendments

- **2026-10-18, proposed.** [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]], §7 amends §5. An explicit item list, confirmed by a person in the app's confirmation sheet or named at a terminal with `ddc clean --items`, counts as interactive confirmation for each listed item, `review` and `native-only` included.
  - **What it changes.** Until now no non-interactive call could remove a `review` item, because `--yes` refuses that class. With `--items`, a script without a TTY can move named `review` items to the Trash, and, with `permanent: true` on an item, delete them permanently.
  - **What bounds it.** Consent is per item: a global `--permanent` with `--items` is refused, and each item's class and action must match the fresh scan, or it is skipped as changed since the scan. Everything else in §5 still applies to every item: the rescan, the re-checks and the refusals. In the app, the confirmation sheet's Clean action is the only caller of `clean`, and a test enforces it.
  - §8 is unchanged: the background path still never calls the executor.
  - The amendment takes effect when that ADR is accepted.
- **2026-10-20, decided by the owner.** Story "User data: pick your own files for the Trash" (epic DDC-002) amends §1, §2 and §5. The user's own files may enter a plan when the user picks them one by one in the app's Review window, or names them in an `--items` list, and only to the Trash.
  - **What it changes.**
    - **§1.** The `user-data` row's "never enters the plan" becomes: never by default, and an entry of a user-data folder can be picked by hand, to the Trash only.
    - **Entries.** An entry is a direct child (for backups, one backup folder) of a finding of `user.downloads`, `user.pictures`, `user.movies`, `user.music`, `user.ios-backups` or `user.imazing-backups`.
    - **The `none` method** still covers the folders themselves, and every other `none` finding: messenger containers, `user.kept`, cloud folders (still not traversed) and the Trash. The executor relaxes `none` only for an entry of those six findings.
  - **What can be picked.** An entry the fresh scan lists, unlocked: a regular file or a directory, never a symlink, another kind or a dataless entry.
  - **Locked entries** are listed and never picked:
    - entries managed by an app (library packages and fixed paths, from a maintained list);
    - entries holding a `.ddc-keep`;
    - protected folders (§6);
    - entries holding a mount point;
    - entries added or modified in the last 10 minutes, partial downloads, and backups that did not finish.
  - **The marker (§2).** For these six rules, a `.ddc-keep` locks the entry that holds it, and no longer turns the whole folder into `user.kept`.
  - **Re-checks (§5).** For these items, §5's rule-marker checks, written for directories found by rules, are replaced by:
    - the parent is the folder, with the identity of the fresh scan's finding;
    - the entry, opened through the parent's file descriptor, is a regular file or a directory with the listed `(dev, ino)`;
    - the fresh scan still holds it as an unlocked entry, checked against the folder's full list of entries, not the report's top 100;
    - every other §5 check still runs: inside the scan roots, no symlink above the parent, protected paths, in use, `.ddc-keep`. Only four are replaced: the rule-marker match, the `.git` or project-root check, the `incomplete` check and the directory-only check.

    The entry runs before any finding inside it, and that finding is skipped as `inside-another-item` only when the entry went to the Trash. The new skip reasons come with a raised clean-report schema version. The in-use check does not see open files: an open file can reach the Trash, and the Trash keeps it.
  - **What bounds it.**
    - The action is `trash`: an `action: delete` item is skipped as `action-not-allowed`, and `permanent: true` refuses the whole list when it is read (exit 2).
    - `--permanent` with `--items` is refused by the argument parser.
    - `--yes` and `--class` never select user data, and nothing is picked by default.
    - §8 is unchanged: the background path never calls the executor.
  - **The cost accepted.** One `--items` run can send many large entries, whole device backups included, to the Trash. The Trash is the only undo, and the space comes back when the user empties it. The mitigation is in the app: no default selection, no Select all for user data, and a "Your own files" block that comes first in the confirmation sheet.

## References

- [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]]
- [[adr/rust-core-library-and-cli-with-json-output|Rust core: library + CLI with JSON output]]
- [[adr/menu-bar-app-swift-shell-over-the-ddc-cli|Menu bar app: Swift shell over the ddc CLI]]
- Cache Directory Tagging Specification (bford.info/cachedir)
- Rust `std::fs::remove_dir_all` (CVE-2022-21658)

---

*Last updated: 2026-10-20*

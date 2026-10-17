---
type: research
slug: "where-developer-disk-space-goes-macos"
title: "Where a developer's disk space goes (macOS)"
status: final
date: 2026-10-17
authors: ["Evgenii Konev"]
tags: [disk, macos, dev-artifacts, cleanup]
review_status: reviewed
reviewed_at: 2026-10-17
---

# Where a developer's disk space goes (macOS)

## Question

Where disk space is lost on a developer's machine, which categories the "junk" falls into, by which signs it can be found reliably, and which of it can be deleted without loss. The answer defines the rule catalog and the risk model for `ddc`, a CLI cleaner that will later become a Mac app.

## TL;DR

On the reference machine (926 GB, 807 GB used, 84 GB available per `df`; the difference is APFS purgeable space), about **150–180 GB** of the 534 GB in `$HOME` belongs to dev categories: project environments and dependencies (`.venv` ≈ 19.5 GB, `node_modules` ≈ 18.5 GB), package manager caches (≈ 30 GB), VMs and containers (≈ 57 GB, five runtimes installed at the same time), toolchains and SDKs (≈ 22 GB), ML models, IDE data. An ordinary "cleaner" does not see this, because it needs semantics: a `.venv` is recognized by `pyvenv.cfg`, not by its name; `target/` is deleted only next to `Cargo.toml`; and the Docker VM image contains volumes with databases. In addition, `du` overstates the space that can be freed: on macOS both uv and pnpm **clone** files (APFS clonefile) from their cache/store by default. Measurement on `~/Projects`: 120.6 GiB allocated; deleting each file separately would free 99.5 GiB. The tool needs four risk classes and a count of the space that is **actually freed**.

## Method

Measurement on 2026-10-17, read-only, without sudo:

- `du -x -d 1 -k $HOME`, `du -x -d 2 -k ~/Library`, targeted `du -sk` of known dev directories;
- `find $HOME -xdev -prune` over artifact markers (`node_modules`, `.venv`, `venv`, `env`, `target`, `build`, `dist`, `__pycache__`, `.next`, `.tox`, `Pods`, `DerivedData`, `.gradle`), pruning `~/Library` and `~/.Trash`, then `du -sk` on every path found;
- `brew cleanup -n`, `tmutil listlocalsnapshots /`, `docker system df`, `xcrun simctl list`.

Limitations: system regions (`/Library`, `/private/var`) are not fully counted without sudo; sizes from separate `du` calls do not deduplicate hard links and APFS clones across directories; the Docker daemon was not running, so `docker system df` is empty; a shell without Full Disk Access gets `EPERM` on `~/Library/Safari`, `~/.Trash`, `~/Library/Application Support/MobileSync`, `~/Library/CloudStorage`.

In addition, a Rust walk prototype (`rayon`, 10 cores) over `~/Projects` (2.24 million files, 233 thousand directories, 120.6 GiB; the total matches `du -skx` byte for byte):

| Walk method | Time |
|---|---|
| `du -skx` | 47 s (cold and warm runs barely differ) |
| `std::fs::read_dir` + `lstat` on every entry, in parallel | 14.3 s |
| `getattrlistbulk` (a batch of entries per system call), in parallel | 6.8–8.8 s |
| the same + `ATTR_CMNEXT_PRIVATESIZE` for every file | 22 s |

The data snapshot (paths, sizes, expected rule and class) is `bench/baseline-2026-10-17.tsv`; the epic's completeness criterion is measured against it. It lists personal folder names, so it stays on the reference machine and is not committed; `bench/make_baseline.py` builds the same snapshot on any machine.

## Findings

### 1. Overall picture

| Region | Size |
|---|---|
| All of `$HOME` | 534 GB |
| `~/Library` | 216 GB (Application Support 123, Caches 49, Containers 21, Group Containers 13) |
| `~/Projects` | 120 GB |
| `~/Pictures` | 75 GB (photos are user data) |
| `~/.local` | 28 GB (of which `~/.local/share/containers`, podman, is 25 GB) |
| Other dot directories | `~/.cache` 12, `~/.npm` 11, `~/go` 10, `~/.ollama` 9, `~/.colima` 8, `~/.lima` 4.5 GB |

About 270 GB is used outside `$HOME`: applications, Homebrew, system regions.

### 2. Project artifacts (8,817 directories found)

| Marker | Count | Size | How to recognize reliably |
|---|---|---|---|
| `.venv` / `venv` / `env` | 76 / 26 / 28 | 19.5 / 0.3 / 0.0 GB | the directory contains `pyvenv.cfg` (the name guarantees nothing: `env/` can be an ordinary folder) |
| `node_modules` | 279 | 18.5 GB | `package.json` is next to it; nested `node_modules` inside `node_modules` are not counted separately |
| `build` | 206 | 8.0 GB | **ambiguous**. A counterexample from this machine: `GrammarHelper/GrammarHelper/build` (7.8 GB) sits next to `*.xcodeproj` but holds experiment results (`.txt`, `.json`, `.jsonl`, `.csv`), not in git, so it cannot be restored. Xcode builds into DerivedData by default, so a `*.xcodeproj` next to it is **not** a sign of an artifact. Reliable markers: `CMakeCache.txt` inside the directory; for Gradle, `build.gradle*` next to it **and** `intermediates/`, `classes/` or `tmp/` inside |
| `dist` | 361 | 2.4 GB | ambiguous, often these are published artifacts; report only by default |
| `__pycache__` | 7,830 | 1.1 GB | always regenerated |
| `target` | 8 | 0.03 GB | `Cargo.toml` is next to it (for Maven, `pom.xml`) |
| `.next` | 2 | 0.06 GB | `next.config.*` is next to it |

A typical case: ten git worktrees of one project (`md-epic003`, `md-epic022`, `md-pair`, `md-lint`, etc.), each with its own `.venv` ≈ 1.1 GB, that is about 11 GB spent on copies of one environment. Hence the requirements: group findings by project and show activity (project mtime or last commit), so that abandoned worktrees are visible at once.

### 3. Package manager caches (they restore themselves)

`~/.npm` 11.4 GB · `~/.cache/uv` 6.9 GB · `~/Library/Caches/pip` 5.6 GB · `~/Library/Caches/pypoetry` 1.2 GB · `~/Library/Caches/Homebrew` 1.1 GB (+ `brew cleanup` will free another ~1.1 GB) · `~/.pnpm-store` + `~/Library/pnpm` 1.4 GB · `~/Library/Caches/ms-playwright` 1.6 GB · `~/go/pkg/mod` 10.4 GB. About 30–40 GB in total.

Almost every manager has a native cleanup command (`npm cache clean --force`, `uv cache clean`, `pip cache purge`, `brew cleanup`, `go clean -modcache`, `pnpm store prune`). Some of them are useful: `uv cache clean` takes the cache lock, `go clean -modcache` handles read-only module directories. But not all of them do what is needed to free space: `uv cache prune` and `pnpm store prune` remove only unused entries, `brew cleanup` also removes old formula versions in `/opt/homebrew`, and `yarn cache clean` for Berry works only inside a project. So the native command is chosen per rule, not used by default; otherwise the directory is deleted.

### 4. VMs and containers

Docker Desktop 17.4 GB (`~/Library/Containers/com.docker.docker`), podman 25.4 GB, colima 8.4 + 1.3 GB, lima 4.5 GB, rancher-desktop 1.5 GB, ≈ 57 GB in total. Five container runtimes are installed at the same time. Each keeps everything in a VM image: for example, `~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw` is one sparse file (64 GiB logical, 17.4 GB allocated) with images, containers **and volumes**. Deleting the directory or the file destroys the databases in the volumes. Only native cleanup with the daemon running and an explicit target is safe (`docker --context desktop-linux builder prune -f`, `docker --context desktop-linux image prune -f`, `podman image prune -f`). `system prune` removes stopped containers together with their data and is excluded. Only a person can decide to remove a runtime entirely, and does it by hand.

### 5. Toolchains, SDKs, models

`~/.nvm` 2 GB (several Node versions), `~/.rustup` 1.1 GB, `~/Library/Arduino15` 7.2 GB, `~/.platformio` 0.5 GB, `~/.ollama` 8.9 GB (LLM models), `~/.cache/pyserini` 2.6 GB (indexes), `~/.cache/codex-runtimes` 1.6 GB. All of this can be restored, but at a cost: gigabytes of downloads, and sometimes specific versions are needed. The class is "at the user's discretion".

### 6. IDEs and editors

JetBrains: Caches 7.8 + Application Support 7.4 GB (data of old IDE versions usually lives there). Editor extensions and data: `~/.vscode` 3.5, `~/.vscode_123` 2.6, `~/.cursor` 2.9, `~/.windsurf` 0.8 GB, plus Application Support of Cursor, Code and Zed at 2–3 GB each. Caches (`Cache`, `CachedData`, `GPUCache`, `logs`) are safe. `workspaceStorage` and extensions are at the user's discretion.

### 7. Ordinary application caches

Adobe Camera Raw 7.9 GB, Spotify 3.7, Google 3.7, Brave 1.6, Firefox 1.1 GB. Safe if the application is closed.

### 8. User data (the tool does not delete it, only shows it)

a photo library of 75 GB, videos kept inside projects 19.6 GB, iPhone backups 17.5 GB, games 48.4 GB, a messenger's data 8.5 GB, an AI assistant app's data 12.6 GB, `~/Downloads` 7.7 GB, old datasets 8.1 GB. "Large files", "not opened for a long time" and "duplicates" are useful here, but the decision is always the person's.

### 9. Counting: `du` ≠ freed space

- **APFS clones.** uv (`link-mode`, `clone` by default on macOS) and pnpm (`package-import-method=auto`: clone → hardlink → copy) install packages through clonefile. On this machine, files in `node_modules/.pnpm` have `nlink = 1`, so they are clones, not hard links. A clone's blocks are shared with the cache, so deleting a venv frees less than `du` shows. `getattrlist`/`getattrlistbulk` with `ATTR_CMNEXT_PRIVATESIZE` (since macOS 10.15) returns the bytes that are freed **immediately** when the file is deleted: not shared with a clone and not held by a snapshot. This is a lower bound; allocated blocks are the upper bound. If the venv and the uv cache are deleted together, more is freed than the sum of their private sizes. An exact count of the union requires grouping by `ATTR_CMNEXT_CLONEID`.
- **Hard links.** Rare (pnpm only with `package-import-method=hardlink`), but they occur. Within one walk, deterministic deduplication by `(dev, inode)` is needed.
- **Trash.** Moving to the Trash on the same volume is one `rename(2)`, instant for any number of files. But the space is not freed until the Trash is emptied.
- **Local Time Machine snapshots.** They hold deleted blocks (then private size ≈ 0); the reference machine has none, but the tool must check `tmutil listlocalsnapshots /` and warn.
- **File Provider (iCloud Drive, Google Drive in `~/Library/CloudStorage`).** A walk can materialize "dataless" files and directories over the network. The scanner must set `setiopolicy_np(IOPOL_TYPE_VFS_MATERIALIZE_DATALESS_FILES, IOPOL_SCOPE_PROCESS, IOPOL_MATERIALIZE_DATALESS_FILES_OFF)` and must not enter these regions.

## Comparison

Existing tools (based on their documentation and project READMEs):

| Option | Pros | Cons |
|---|---|---|
| `npkill` | finds `node_modules` fast, interactive selection | Node only; does not see global caches |
| `kondo` (Rust) | project artifacts of many ecosystems | does not see global caches, VMs, IDEs; no background scan and no activity report |
| DevCleaner for Xcode | knows Xcode precisely (DerivedData, DeviceSupport, archives) | Xcode only |
| `dust`, `dua-cli`, `ncdu`, DaisyDisk, GrandPerspective | fast size visualization | do not know what exactly they are looking at or whether it is safe to delete |
| CleanMyMac and similar | general caches and applications, has a GUI | paid; weak at recognizing dev artifacts; the risk class is opaque |
| Native commands (`uv cache clean`, `go clean -modcache`, `docker image prune`…) | know how their own region is laid out | there are dozens of them, and each has to be remembered; some do more or less than you expect |

No tool combines: semantic recognition per ecosystem, global caches, VMs and IDEs, grouping by project with activity, an honest count of freeable space, a clear risk class and a background report.

## Conclusion

Requirements for `ddc` that follow from the data:

1. **Rules, not names.** Each rule is a marker (a file or a path) plus confirmation of the context (`pyvenv.cfg`, `Cargo.toml` next to it, etc.). Ambiguous names (`build`, `dist`, `env`) are not deleted without confirmation.
2. **Four risk classes:** `safe` (a cache, restores itself), `rebuildable` (a project artifact, restored by an install or a build), `review` (expensive to restore: VMs, models, toolchains, backups), `user-data` (report only).
3. **Grouping:** risk class → category → project, with activity inside each group.
4. **Honest counting from the first version:** "allocated" (blocks, deterministic hard-link deduplication) and "frees at least" (APFS private size) where clones are expected (uv, pnpm, bun); for other findings, freeable ≈ allocated, and the total is an estimate.
5. **The native cleanup command comes first** where one exists; for VMs and containers it is the only allowed method.
6. **Walk via `getattrlistbulk`:** twice as fast as `std::fs` and 5–7 times faster than `du`.
7. **Background mode only scans** and notifies (user decision of 2026-10-17).

The stack and safety-model decisions are recorded in separate ADRs.

## References

- Measurement data snapshot: `bench/baseline-2026-10-17.tsv` on the reference machine (not committed: it lists personal folder names); generator `bench/make_baseline.py`
- Apple: `getattrlist(2)`, `ATTR_CMNEXT_PRIVATESIZE`
- uv: link-mode (`clone` by default on macOS)
- pnpm: `package-import-method` (`auto`: clone → hardlink → copy)

---

*Last updated: 2026-10-17*

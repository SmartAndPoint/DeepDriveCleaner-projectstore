---
type: research
slug: "where-developer-disk-space-goes-linux"
title: "Where a developer's disk space goes (Linux)"
status: final
date: 2026-10-31
authors: ["Evgenii Konev"]
tags: [disk, linux, dev-artifacts, cleanup]
review_status: reviewed
reviewed_at: 2026-10-31
---

# Where a developer's disk space goes (Linux)

## Question

Which of the macOS findings in [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]] hold on Linux, what Linux adds, and what the Linux section of the catalog needs before story "Linux scanner and rule catalog" (epic DDC-007) flips Linux to supported.

## TL;DR

Two Linux machines in daily developer use were measured, read-only. The developer caches the shared catalog already knows sat where its Linux paths say (`~/.cache/pip`, `~/.cache/pypoetry`, `~/.cache/uv`, `~/.cache/huggingface`, `~/.rustup/toolchains`), and they were small: a few gigabytes per machine at most. What Linux adds:

1. **Remote editor servers.** VS Code and Cursor install a server per editor version into `~/.vscode-server` and `~/.cursor-server` on every machine they reach over ssh. The current layout keeps at most five builds (about 1 GB per editor) and evicts the rest itself. What it never removes are the `.staging` folders of interrupted installs, and builds in the older `bin/<commit>` layout.
2. **Folders inside the home that the user does not own.** Containers write into bind-mounted project folders as root. Some of these folders are unreadable (`0700`, a database's data); others are readable build output owned by root. A walk meets both, and a delete cannot remove either.
3. **Root Docker lives outside the home.** On one machine, Docker's images and volumes under `/var/lib/docker` were tens of gigabytes, an order of magnitude more than all developer caches in its home together.
4. **Hard links instead of clones.** ext4 has no clones. A venv that uv built there shares almost all of its files with the uv cache through hard links.

## Method

Measured on 2026-10-31, read-only, as each machine's ordinary user, without sudo:

- `du -sxh` over the home's top level, `~/.cache/*`, `~/.local/share/*`, `~/.config/*` and the known tool folders;
- `find -maxdepth 5 -prune` over artifact markers (`target`, `node_modules`, `.venv`, `venv`, `build`, `dist`, `.next`, `.gradle`, `__pycache__`), then `du -sxh` on each path found;
- `find -type f -links +1` inside each venv found by `pyvenv.cfg`;
- `docker system df` and `docker info` (`DockerRootDir`);
- `env` for `XDG_*_HOME` overrides.

Both homes are on ext4. The machines are private, so this note describes what the tools did on them, not the machines. The raw measurements stay with the owner, as the macOS note's baseline does.

Limitations:

- No JavaScript, JVM or Go caches were present: `~/.npm`, `~/.gradle`, `~/.m2`, `~/go/pkg/mod`, pnpm, Yarn and bun were absent on both. Their Linux paths in the catalog come from the tools' documentation, not from this measurement. The same holds for every item marked "documented" below.
- Neither machine had Flatpak or Snap user data, anything in the freedesktop Trash, rootless Docker, Podman, Docker Desktop or a JetBrains IDE.
- Both trees are small: `du` walked each in under a second. These numbers say nothing about walk speed at scale. The Linux load thresholds come from the synthetic load trees, as the story says.
- Sizes from separate `du` calls do not deduplicate hard links across folders.
- Neither home was on NFS or CIFS (see the open questions).

## Findings

### 1. Package caches: where the shared rules expect them

| Cache | Largest seen | Rule |
|---|---|---|
| `~/.cache/pip` (almost all in `http`) | 1.6 GB | `cache.pip` |
| `~/.cache/pypoetry/{cache,artifacts}` | 0.8 GB | `cache.poetry` |
| `~/.cache/huggingface/hub` | 0.5 GB | `model.huggingface` |
| `~/.cache/uv` | 0.2 GB | `cache.uv` |
| `~/.rustup/toolchains/*` | 1.5 GB (one toolchain) | `toolchain.rustup` |

Every path lay inside the catalog's Linux path for its rule. One caveat: `model.huggingface` matches all of `~/.cache/huggingface`, which also holds the auth files `token` and `stored_tokens`, so the rule would trash the token with the models. This holds on macOS too; the rule should match `hub` (and `xet`, `datasets`) instead.

No `XDG_CACHE_HOME` was set. The catalog joins every path onto the home as a fixed string (`home.join(".cache/...")`). pip, Poetry, uv, Hugging Face and Go honour `XDG_CACHE_HOME`; rustup does not (it reads `RUSTUP_HOME`), and Yarn 1 and Puppeteer need checking per tool. Supporting XDG therefore means a cache-root token per rule, resolved through `env::Env`, and the protected `.cache` entry following the same token. It is a real change to the catalog's path model, not a one-line default.

### 2. Remote editor servers (new)

VS Code Remote-SSH and Cursor (a fork of it) install their server on the machine they connect to.

| Folder | Seen | Who removes it |
|---|---|---|
| `~/.cursor-server/cli/servers/Stable-<commit>` | five builds, 194–215 MB each | the CLI: `cli/servers/lru.json` lists builds most recently used first, and the CLI deletes the oldest once there are more than five (`KEEP_LRU = 5` in VS Code's `cli/src/download_cache.rs`) |
| `~/.vscode-server/cli/servers/Stable-<commit>` | three builds, 165–174 MB each | the same |
| `.../cli/servers/<name>.staging`, `<name>.staging-<id>` | one per editor, 172–178 MB | nobody: the CLI clears a staging folder only when it installs that same commit again, which never happens after an update |
| `~/.cursor-server/bin/<commit>` (older layout, still used when `remote.SSH.useExecServer` is off) | two builds, 214 MB each | not by the CLI; some extension versions clean old installs, unverified for current ones |
| `~/.{cursor,vscode}-server/data` | 75–164 MB | `data/logs` and `data/CachedExtensionVSIXs` are cache candidates; `data/User` (settings, extension state and tokens) is user configuration |
| `~/.{cursor,vscode}-server/extensions` | 0.2–0.6 GB | user configuration, out of scope |

Documented, not measured: `~/.vscode-server/bin/<commit>`, `~/.vscode-server-insiders` and `Insiders-*` builds use the same layouts.

So the space a rule can win here is bounded: the staging leftovers, the old `bin/` builds, and at most four of the five LRU builds, about 1 GB per editor. Its keep must follow the editor's own signal, not "newest": the head of `lru.json`, read the way `toolchain.rustup` reads `settings.toml`, plus every build the in-use check finds live. Two clients on different editor versions connect to different commits at once, so "the newest build is the live one" is false. The sibling-scope keep that `ide.jetbrains-old-data` uses is the existing mechanism to build on. The in-use check needs `/proc/*/exe` (the server's `node` is a real file inside the build) and `/proc/*/fd`: a `.staging` folder of an install in progress has open files and no running binary.

### 3. Folders the user does not own (new)

A container that writes into a bind-mounted project folder writes as root. Two cases matter:

- **Unreadable data.** A database container's data folder, owned by root with mode `0700`. The user cannot list it: `du` and `find` print `Permission denied` and count nothing below it.
- **Readable root-owned output** (documented for build containers, not seen here): `target/`, `node_modules/` or `build/` written as root with modes `0755`/`0644`. A rule matches them and the scan reads them in full.

Neither can be deleted by the user. Today's delete renames a finding to a tombstone first and only then removes it (`exec/delete.rs`). On a root-owned tree, the removal then fails and `make_dirs_writable` cannot chmod root's folders, leaving a `.ddc-deleting-*` tombstone the user cannot remove. A Trash move "succeeds" into a Trash entry the user can never empty. For the unreadable case the rename also moves live database data out from under its mount before anything fails.

So the scanner needs two conditions, both checked before the plan and before any rename:

- An unreadable folder is reported as unreadable, with its path and an unknown size. It is never a finding and never treated as empty, and no rule plans a folder whose subtree it could not read in full.
- A finding with any entry whose owner (`stx_uid`) is not the scanning user is refused with that reason: listed, never planned.

### 4. Root Docker, rootless Docker, Docker Desktop

With root Docker, images, containers and volumes live in `/var/lib/docker`, owned by root. The user reaches them only through the daemon, as a member of the `docker` group. On one machine they came to tens of gigabytes. Most of it was images Docker itself counted as reclaimable, plus volumes no container referenced.

The macOS rule holds on Linux: only native commands with an explicit target are safe (`docker image prune`, `docker builder prune`). Never `docker system prune` or `docker volume prune`: a volume that no container references can still hold a database. The walk never enters `/var/lib/docker`; it is outside the home. Whether `ddc` reports Docker's own numbers on Linux is a catalog decision for the story.

Documented, not measured, and each the analogue of a macOS VM rule:

- rootless Docker keeps its data in `~/.local/share/docker` (no rule today);
- Podman keeps it in `~/.local/share/containers` (`vm.podman` exists);
- Docker Desktop for Linux keeps its VM in `~/.docker/desktop` (no rule today).

All three are the VM class: reported, never deleted.

### 5. User data

The largest folder on one machine was one project's own data, two orders of magnitude above its caches. It is user data: reported, never planned.

The other machine had a Python venv at the top of the home that the shell activates at login (`include-system-site-packages = true`). The `python.venv` rule recognizes it by `pyvenv.cfg`. It is the hardest in-use case on Linux:

- `/proc/<pid>/exe` is the executable after symlinks are resolved, and a venv's `bin/python` is a symlink to the base interpreter, so `exe` never points into a venv. The macOS check reads the path before resolution (`KERN_PROCARGS2`, safety ADR §5) for exactly this reason.
- `/proc/<pid>/environ` is a process's initial environment. A shell that sources `activate` from its startup file has no `VIRTUAL_ENV` in its own `environ`; only the processes it starts afterwards do.

What does see a venv in use on Linux: the initial `environ` of the shell's children, `cmdline` when `argv[0]` carries the venv's path, `cwd`, and `/proc/*/fd` and `/proc/*/maps` for files under the venv's `site-packages`. An idle shell with an activated venv and no children is invisible to all of them. The Linux in-use check should state that limit rather than claim the macOS semantics.

### 6. Coding agents

`~/.local/share/claude/versions` held two Claude Code builds (474 MB). Whether the native installer prunes this folder itself was not checked. The coding agents' data covers Cursor's agent CLI versions (`~/.local/share/cursor-agent`) but not this folder, on either platform. That gap belongs to the coding agents' data, not to a Linux rule.

### 7. Counting

- **No clones on ext4.** uv 0.8.22, the measured release, defaulted to `hardlink` on Linux. Current uv defaults to `clone` and falls back to `hardlink` where the filesystem cannot reflink, as ext4 cannot. On ext4, every uv version therefore hard-links; on btrfs and XFS, current uv makes reflinks, which behave like APFS clones. The `private-size` capability is absent on Linux, as the story states. The clone markers the macOS rules carry (uv venvs, pnpm's `node_modules`) trigger nothing on Linux. The shared `cache.pnpm` hint that cloned files "stay intact" is macOS wording and needs a Linux variant.
- **Hard links are the Linux signal.** A uv-made project venv (uv 0.8.22) had 2,621 of its 2,851 files with a link count above one, shared with the uv cache. Venvs made by `python -m venv` and pip had none. Deleting such a venv frees almost nothing while the cache lives. The scan needs deduplication by `(stx_dev, stx_ino)`, and the report should say when a finding's files have links outside it: an `stx_nlink` above the number of links the walk saw inside the finding.
- **Trash.** The freedesktop Trash on the home's filesystem is a rename into `~/.local/share/Trash/files` plus a `.trashinfo` file. As on macOS, nothing is freed until the Trash is emptied. On another filesystem the spec tries `$topdir/.Trash/$uid` (an admin-created, sticky folder), then `$topdir/.Trash-$uid`. Whether a move that finds neither fails or copies is the `trash` crate's choice, to be checked.

### 8. Not found, and decided

- **Cloud sync.** None was present. Dropbox, Insync and the Nextcloud client keep real files on disk, so they are user data. rclone, sshfs and gocryptfs mounts are other filesystems, which the one-filesystem walk (`stx_dev`) never enters. Nextcloud's Linux virtual files are small placeholder files with a suffix. None of these materializes data when walked, so no substitute for the absent `dataless-files` capability is needed.
- **Flatpak and Snap.** `~/.var/app/<id>/cache` and `~/snap/<name>/<rev>` (documented) hold app caches and old snap revisions' user data. They are deferred out of this story: neither was present, and the first release targets developer tools.
- **JetBrains** (documented). `~/.cache/JetBrains/<product><version>` is the analogue of the macOS caches. `~/.local/share/JetBrains/<product><version>` is the analogue of the old-version data that `ide.jetbrains-old-data` covers. Gateway's remote IDE backends under `~/.cache/JetBrains/RemoteDev/dist/` are the JetBrains counterpart of §2, at 1–3 GB each. The macOS JetBrains rules are macOS-only (a rule with `owners`, macOS app-bundle names, stays in the macOS section), so Linux needs its own JetBrains rules or per-OS owners.

## Comparison with macOS

| | macOS | Linux |
|---|---|---|
| Cache root | `~/Library/Caches` and `~/.cache` | `$XDG_CACHE_HOME`, default `~/.cache`, honoured per tool |
| Shared blocks | APFS clones (uv, pnpm); private size needed | hard links on ext4; reflinks on btrfs and XFS; no private size |
| Containers | VM image in the home | root Docker outside the home; rootless, Podman and Docker Desktop in it |
| Folders the user cannot read or delete | TCC-protected, a fixed set | arbitrary, created by root containers |
| In-use by executable | path before symlink resolution | `/proc/*/exe` is after resolution; venvs need `fd`, `maps`, `cmdline` |
| Other users' processes | single-user in practice | shared machines are common; the check sees only one's own processes |
| Remote editor servers | rare | on any machine a developer reaches over ssh |

## Conclusion

For the Linux section of the catalog and the Linux scanner:

1. **Keep the shared rules' Linux paths.** Every measured cache matched. XDG support is a per-rule cache-root token through `env::Env`, for the tools that honour it; it is the story's call whether it lands with the flip.
2. **Add a rule for remote editor servers.** It covers the `.staging` and `.staging-*` leftovers, the old `bin/<commit>` builds and the non-head `cli/servers` builds. It keeps the head of `lru.json` and every build held live by `/proc/*/exe` or `/proc/*/fd`. Class `safe`: the editor reinstalls a missing build on connect.
3. **Refuse folders the user cannot read or does not own** before planning and before any rename. This is a scanner rule with tests, not a catalog entry.
4. **Never walk `/var/lib/docker`.** Add VM-class rules for rootless Docker and Docker Desktop for Linux in the home.
5. **Count hard links honestly:** deduplicate by `(dev, ino)` and flag findings whose files have links outside them.
6. **State the Linux in-use limits:** `exe` covers real binaries only; venvs rely on `fd`, `maps`, `cmdline`, `cwd` and children's `environ`; other users' processes are invisible.
7. **Narrow `model.huggingface`** so the auth token is never trashed, on both platforms.
8. **Add Linux JetBrains rules** (own rules or per-OS owners); defer Flatpak and Snap.
9. **Follow-up outside this story:** Claude Code's `~/.local/share/claude/versions` belongs in the coding agents' data, reported on both platforms.

## Open questions

- NFS and CIFS homes (common on university and corporate machines): uid mapping, `stx_dev` and latency were not measured.
- The uv release in which the Linux default moved from `hardlink` to `clone`.
- Whether current Remote-SSH still removes old `bin/<commit>` installs itself.
- Whether Claude Code's native installer prunes its `versions` folder.

## References

- XDG Base Directory Specification (`XDG_CACHE_HOME`, `XDG_DATA_HOME`, `XDG_STATE_HOME` and their defaults)
- freedesktop.org Trash specification 1.0 (home trash; `$topdir/.Trash/$uid`, then `$topdir/.Trash-$uid`)
- VS Code CLI: `cli/src/download_cache.rs` (`lru.json`, `KEEP_LRU = 5`, staging folders), `cli/src/tunnels/paths.rs`
- uv: settings reference, `link-mode`
- proc(5): `/proc/<pid>/exe`, `environ`, `fd`, `maps`
- Docker: `docker system df`, `docker image prune`, `docker builder prune`

---

*Last updated: 2026-10-31*

---
type: story
id: "story-linux-catalog-flatpak-snap-and-the-tools-only-macos-lists"
epic: "DDC-010"
title: "Linux catalog: Flatpak, Snap and the tools only macOS lists"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [linux, rules, catalog, research]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Linux catalog: Flatpak, Snap and the tools only macOS lists

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

On 2026-11-01, `ddc rules --platform macos` listed 89 rules and `ddc rules --platform linux` listed 74: 16 rules are macOS-only, and one (`vm.docker-rootless`) is Linux-only. The 16 macOS-only rules:
- `app.adobe-camera-raw-cache`, `cache.cocoapods`, `cache.colima`, `cache.homebrew`;
- `macos.app-bundle`;
- `user.cloud`, `user.imazing-backups`, `user.ios-backups`, `user.telegram`, `user.whatsapp`;
- `vm.orbstack`, `vm.rancher-desktop`;
- `xcode.archives`, `xcode.derived-data`, `xcode.device-support`, `xcode.simulator-caches`.

Most of them have no Linux subject. Some do:
- Colima runs on Linux, with its cache in `~/.cache/colima`.
- Rancher Desktop keeps its data in `~/.local/share/rancher-desktop`.
- Telegram Desktop keeps its data in `~/.local/share/TelegramDesktop`.
- Homebrew's cache on Linux is `~/.cache/Homebrew`; story "Homebrew on Linux: ddc disk brew-packages and its cleanup" takes that rule.

Linux also has its own app formats and caches, which the research note [[research/where-developer-disk-space-goes-linux|Where a developer's disk space goes (Linux)]] deferred (§8):
- **Flatpak**:
  - each app's cache in `~/.var/app/<id>/cache`;
  - user-installed runtimes in `~/.local/share/flatpak`, often several GB, that no app uses any more.
- **Snap**: user data per revision in `~/snap/<name>/<rev>`. snapd keeps old revisions, and each keeps its own copy of the user data.
- **Steam**: `~/.local/share/Steam`, also reached as `~/.steam/steam` (a symlink) or inside the Flatpak app (`~/.var/app/com.valvesoftware.Steam/.local/share/Steam`), with `steamapps/shadercache` (a cache) and `steamapps/compatdata` (Proton prefixes, which hold save games). It is the largest desktop cache on many Linux machines.
- **Desktop caches**:
  - thumbnails (`~/.cache/thumbnails`);
  - GPU shader caches: `~/.cache/mesa_shader_cache` and, from Mesa 24, `~/.cache/mesa_shader_cache_db`, plus `~/.nv/ComputeCache` and `~/.cache/nvidia`.
- **Developer caches no rule covers on any OS**:
  - Bazel's output bases (`~/.cache/bazel`);
  - ccache (`~/.cache/ccache`) and sccache (`~/.cache/sccache`);
  - Deno (`~/.cache/deno`);
  - Electron's downloads (`~/.cache/electron`);
  - node-gyp headers (`~/.cache/node-gyp`);
  - pre-commit's environments (`~/.cache/pre-commit`);
  - the Android SDK and its emulator images (`~/Android/Sdk`, `~/.android/avd`).

  macOS has most of these too, under other paths, so they become shared rules with per-OS paths.

The catalog is evidence-first, as the macOS and Linux catalogs were. Each rule needs a measured or documented size, a risk class and the native clean command where one exists.

## Decomposition

- [ ] A research section in [[research/where-developer-disk-space-goes-linux|Where a developer's disk space goes (Linux)]], "Flatpak, Snap and the caches no rule covers". For each candidate it gives:
  - the path on Linux and on macOS;
  - a size, measured where the owner's machines or a seeded VM have it and documented otherwise;
  - who recreates it and how;
  - the native clean command: `flatpak uninstall --unused --user` for unused user runtimes, `snap remove --revision` (root, so printed only), `bazel clean --expunge`, `ccache -C`, `deno clean`. sccache has no clean command (`--zero-stats` clears statistics only);
  - the risk class.
  
  Run the critic on the section.
- [ ] Rules, each with its fixture test (`rule_path`) and a cases-table entry where a new command form appears:
  - Flatpak app caches: class `safe`. Each `~/.var/app/<id>/cache` is a known child; the app's `data` and `config` are never matched.
  - Flatpak's unused user runtimes in `~/.local/share/flatpak`: class `review`, cleaned only by the native `flatpak uninstall --unused --user`, never deleted by path. A scan never runs `flatpak`. The research section settles how the dry run lists what would go, and whether it can only say "flatpak decides".
  - Snap's old revisions: the user data of a revision that is not `current` (`~/snap/<name>/current` is a symlink to the live one). Class `review`, since it is the old revision's user data, not a cache.
  - Steam's shader cache: class `safe`. `compatdata` is user data and is never planned.
  - `cache.colima`, `vm.rancher-desktop` and `user.telegram` gain Linux paths. Colima's VM disks stay VM class.
  - The desktop and developer caches the section approves.
- [ ] `ddc rules --platform linux` and its golden change. Update the rules count in the README and on the website, if they name it.
- [ ] In-use check: Flatpak and Snap apps run in their own mount namespace. Check that `/proc/<pid>/fd` and `cwd` of a sandboxed app show paths the check can compare. If they do not, the rule's class says so, and the clean refuses while the app runs. Use the app id in the process's `cgroup`, or `flatpak ps`.

## Implementation Plan

## Acceptance Criteria

- [ ] The research section exists with a size and a class per candidate, critic-approved.
- [ ] `ddc scan` on a seeded Linux home finds a Flatpak app's cache and an old Snap revision's data, and never matches a Flatpak app's `data` or `config` or the `current` Snap revision (tests).
- [ ] Steam's shader cache is found, and `compatdata` is never planned (tests).
- [ ] `cache.colima`, `vm.rancher-desktop` and `user.telegram` are listed by `ddc rules --platform linux` (rules golden).
- [ ] A sandboxed Flatpak app that holds a file in its cache keeps that cache out of a clean (test with a fake process entry; an owner check with a real app).

## Final Summary

## Technical Notes

- `~/.var/app/<id>/cache` is bind-mounted into the sandbox. A running app opens files at `/var/cache/…` or at the app's own path inside its namespace. Read the host path through `/proc/<pid>/root`, or match by app id.
- The Snap revision folders are owned by the user. snapd itself removes them with `snap remove --revision`, which needs root. Removing only the user-data copy of an old revision is what `ddc` can do without root, and the research section must say whether snapd minds.

## Dependencies

- Story "Cache roots moved by the environment: XDG and each tool's own variable", for the token, if it lands first.
- Story "Linux mount boundaries: automounts, bind mounts and the Trash across filesystems": Snap mounts squashfs images under `/snap`, outside the home.

## Attachments

-

---

*Last updated: 2026-11-01*

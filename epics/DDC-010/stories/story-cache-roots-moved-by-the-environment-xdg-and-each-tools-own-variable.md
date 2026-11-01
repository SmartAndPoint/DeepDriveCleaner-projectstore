---
type: story
id: "story-cache-roots-moved-by-the-environment-xdg-and-each-tools-own-variable"
epic: "DDC-010"
title: "Cache roots moved by the environment: XDG and each tool's own variable"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [rules, catalog, linux, macos, env]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Cache roots moved by the environment: XDG and each tool's own variable

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

Every catalog rule today names a fixed path under the home. Linux tools honour `XDG_CACHE_HOME` and `XDG_DATA_HOME`. Most tools also honour a variable of their own, on every OS:
- `CARGO_HOME` and `RUSTUP_HOME`;
- `GOPATH`, `GOMODCACHE` and `GOCACHE`;
- `npm_config_cache`, `YARN_CACHE_FOLDER` and the pnpm store dir;
- `PIP_CACHE_DIR`, `UV_CACHE_DIR` and `POETRY_CACHE_DIR`;
- `HF_HOME` and `HF_HUB_CACHE`;
- `GRADLE_USER_HOME`;
- `CONDA_PKGS_DIRS`;
- `OLLAMA_MODELS`;
- `TORCH_HOME`.

A developer who moved a cache, often to a second disk or out of a small home, gets nothing from the scan. A missed root only under-reports, so nothing unsafe happens. But the biggest caches are the ones people move.

Story "Linux scanner and rule catalog" deferred this as "the XDG cache-root token" (research note, Conclusion 1). The probes in the catalog (`go env GOMODCACHE`, `uv cache dir`, `rustup show home`) only check that a native clean command acts on the folder the rule found. They do not find a moved one.

The background scan raises its own question. A LaunchAgent or a systemd user unit does not see the shell's variables. So the interactive scan and the scheduled scan could look in different places.

## Decomposition

- [ ] An `env::Env` field per honoured variable, read once at start, as `TMPDIR`, `DOCKER_CONFIG` and `CLAUDE_CODE_TMPDIR` are today (`crates/ddc-core/src/env.rs`). The catalog's paths gain a root token (XDG cache, XDG data, or a tool's variable) that resolves through `Env`. Default: today's path.
- [ ] Protected folders move with their tokens. The safety ADR §6 protects `~/.cache` and `~/.local` as folders, not as their contents. With `XDG_CACHE_HOME=~/xdg-cache`, the folder `~/xdg-cache` itself must be protected the same way, or the whole cache root becomes plannable. The same holds for `XDG_DATA_HOME`. That variable also moves the freedesktop Trash (the `trash` rule) and the `~/.local/share` rules (opencode, Podman storage, rootless Docker). The research note's §1 asked for this ("the protected `.cache` entry following the same token"). Amend §6 and run the critic on the amendment.
- [ ] A moved root outside the scanned roots.
  - The proposal: a moved cache is scanned as its own extra root only when it lies under the home, or when the user names it with `--root`.
  - A root outside the home is listed in the report as "moved by `<VAR>` to `<path>`, not scanned", with the suggested command that scans it.
  - Decide this, and run the critic on it, before the code. A cache on another disk is outside the one-filesystem walk, and scanning it silently would surprise.
- [ ] Record the roots used in the scan report: variable, value and whether it was scanned. Save them in `last-scan.json`, so a background scan and an interactive one can be compared. Schema version move, docs and DDCKit.
- [ ] The background scan. Story "Scheduled scans on Linux through a systemd user timer" decides first which variables go into `Environment=`. This story applies the same rule to the LaunchAgent's `EnvironmentVariables`, and `schedule status` names them on both OSes.
- [ ] A rule whose native clean command uses a probe compares the probe's answer with the resolved root, not the default path. Today `LocationCompare::Equals` compares with the default.
- [ ] Tests:
  - each honoured variable moves its rule's root, on macOS and Linux;
  - a root outside the home is reported and not scanned;
  - a probe compares against the moved root;
  - a cases-table case runs `ddc scan` with `XDG_CACHE_HOME` set.

## Implementation Plan

## Acceptance Criteria

- [ ] With `XDG_CACHE_HOME=~/xdg-cache`, `ddc scan ~` on Linux finds the pip, uv, npm and JetBrains caches under `~/xdg-cache` (test plus a cases-table case).
- [ ] With `CARGO_HOME`, `GOMODCACHE`, `PIP_CACHE_DIR`, `HF_HOME` or `GRADLE_USER_HOME` set to a folder under the home, the rule finds it there, on macOS and Linux (test per variable).
- [ ] A moved root outside the scanned roots is named in the report with its variable, and is not walked (test).
- [ ] With `XDG_CACHE_HOME` or `XDG_DATA_HOME` moved, the moved folder itself is never planned, and the Trash rule follows `XDG_DATA_HOME` (tests). The safety ADR §6 says so, critic-approved.
- [ ] The scan JSON records the roots used, and the scan schema and DDCKit decode it.

## Final Summary

## Technical Notes

- macOS tools mostly ignore XDG. `pip` and `uv` honour it on Linux only. The token resolves per OS, the same way per-OS paths resolve in the shared catalog today.
- Never run a tool just to learn its root during a scan. Probes run only before a native clean, as today. The variables are read, not evaluated.

## Dependencies

- Story "Scheduled scans on Linux through a systemd user timer", for the environment rule of the background scan. The scan half can land first.
- A root on another disk is cleaned under the cross-filesystem Trash rule of story "Linux mount boundaries: automounts, bind mounts and the Trash across filesystems".

## Attachments

-

---

*Last updated: 2026-11-01*

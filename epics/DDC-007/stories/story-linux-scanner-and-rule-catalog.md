---
type: story
id: "story-linux-scanner-and-rule-catalog"
epic: "DDC-007"
title: "Linux scanner and rule catalog"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [rust, linux, scanner, rules, safety]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Linux scanner and rule catalog

| Field | Value |
|---|---|
| **Epic** | [DDC-007](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

This story fills `platform::linux` and the Linux catalog section, so `ddc scan` and `ddc clean` work on a Linux developer's home with the same risk classes, confirmations and verified-parent deletes as on macOS. Then it flips Linux to supported and publishes the Linux archives.

Before the catalog is written, a research note records where dev disk space goes on Linux, with measured sizes. It plays the role [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]] plays for macOS.

## Decomposition

- [x] Research note [[research/where-developer-disk-space-goes-linux|Where a developer's disk space goes (Linux)]], critic-approved 2026-10-31. Measure on a real or VM Ubuntu developer setup:
  - XDG cache and data (`~/.cache`, `~/.local/share`);
  - per-tool caches (pip, uv, npm, pnpm, yarn, bun, cargo, go, gradle, maven, Hugging Face, JetBrains, VS Code);
  - rootless Docker and Podman storage under `~/.local/share` (VM class: reported, never deleted; root Docker is outside the home);
  - Flatpak and Snap user data.
  
  Run the critic on it.
- [ ] Walker: `getdents64` + `statx` through `rustix`. Allocated size from `stx_blocks`. Hard links via (dev, ino). One filesystem via `stx_dev`. Symlinks never followed.
- [ ] Verified-parent operations (`openat`, `fstatat`, `renameat` to a tombstone, re-check, `unlinkat`) as on macOS, using the same safety ADR §5 contract.
- [ ] Free space and volumes: `statvfs` and `/proc/self/mountinfo`.
- [ ] In-use checks with the same semantics as macOS (safety ADR §5):
  - `/proc/*/fd`;
  - `/proc/*/cwd`;
  - `/proc/*/exe`;
  - `/proc/*/cmdline`;
  - `/proc/*/environ` (own processes only; `VIRTUAL_ENV`).
  
  An unreadable entry counts as "unknown", never as "free".
- [ ] Trash: freedesktop Trash via the `trash` crate, home and other-volume trash.
- [ ] Catalog (Linux section): rules from the research note. Protected folders: the safety ADR §6 list (it already holds `~/.config`, `~/.ssh`, `~/.gnupg`, `~/.local`, `~/.cache` as protected folders, not as their contents), plus Linux-only folders and Linux system folders (`/`, `/home`, `/usr`, `/etc`, `/opt`, `/var`, `/boot`; the shared section from story "Rule catalog split: shared and per-OS sections" has none), amended into the safety ADR §6 before `SUPPORTED` flips (the catalog split's flip-guard test refuses the flip otherwise). The rules JSON gains its `platform` field and schema bump with this flip (seam ADR §4 amendment of 2026-10-30). Linux-only folders: `~/.local/share/keyrings`, `~/.local/share/Trash`, `~/.pki`, browser and mail profiles (`~/.mozilla`, `~/.thunderbird`).
- [ ] Home, data and logs: `HOME`; `$XDG_STATE_HOME/deepdrivecleaner` (default `~/.local/state/deepdrivecleaner`), through `env::Env`.
- [ ] JSON paths: Linux allows names that are not UTF-8, and APFS refuses them. Today `scan --json` would fail outright on one in a reported path (a finding, a listed entry, an error or skipped path, a root or the home; a name strictly inside a claimed finding never reaches the JSON), because serde refuses to serialize a non-UTF-8 `PathBuf`. Decide how the schema represents such a name (lossy with a flag, or escaped bytes), implement it, and check that the `names` load profile, which builds invalid UTF-8 where the filesystem accepts it, passes on Linux.
- [ ] Load thresholds per target: Linux has no bulk stat (one `statx` per entry) and musl's allocator is slow under threads. Measure, then set Linux thresholds; try `mimalloc` for musl if the S check misses by more than 2×.
- [ ] Honour the "when absent" column of ADR "Platform seam: one platform module per OS with declared capabilities", §3, for every capability this platform does not declare: `notifications` (`scan --background` decides but does not notify), `full-disk-access` (`full_disk_access` is true), `private-size` (no `private_bytes`) and `local-snapshots` (`local_snapshots` is 0). Story "Platform seam: ADR and capability model" left them unimplemented: no supported platform lacked one. `dataless-files` is absent on Linux: the catalog lists the cloud-sync roots it does not walk (Dropbox, Nextcloud, Insync and the like), or the research note states that none needs listing.
- [ ] Flip `SUPPORTED` to true for Linux and add the Linux targets to `release.yml`'s enabled list, in one commit.

## Implementation Plan

## Acceptance Criteria

- [ ] The research note exists, critic-approved, with measured sizes.
- [ ] On Ubuntu (CI and a manual VM run), `ddc scan ~ --json` on a seeded home finds the catalog's caches and project artifacts and validates against the schema.
- [ ] DDC-005's property tests pass on Linux: no protected folder, `user-data` item or VM directory is ever planned; symlinks are never followed.
- [ ] A folder swapped for a symlink between scan and clean is skipped as changed since the scan, not followed (test).
- [ ] `ddc clean` moves a selected cache to the freedesktop Trash and the journal records it (integration test in CI).
- [ ] A pre-release tag publishes both Linux archives, and `install.sh` installs them (story "One-line install on macOS and Linux: ddc.sh/install.sh and cargo").

## Final Summary

## Technical Notes

- `ext4` and `btrfs` report allocated size the same way. btrfs reflinks are like APFS clones: private size is out of scope, stated as an absent capability.
- Iterate on a Linux machine with docker through `scripts/native-linux.sh`, not through CI rounds. It runs the Gate's Linux leg in about a minute: clippy, nextest with profile `ci`, doc tests and `scripts/native-skips.sh linux`, in `rust:<toolchain>-bookworm` as a non-root user. Root reads unreadable folders, so it changes results.
- The flip burns down `.config/native-skips/linux.txt`. It held 431 names when the build matrix's stage 2 merged, and it grows with each new test that skips off macOS. After it, only skips through `lacks()` should remain: Homebrew, the whole disk, the coding agents. Plus whatever the research note shows Linux does not have.
- Stage 2 also skipped some tests on Linux that pass there:
  - `there_is_no_yes`;
  - the macOS-path tests (`agents::running`, `agents::liveness`, `free::probe`, `keep`, the malformed-list test);
  - the fuzz regressions' "accepted" half.

  They come back with the flip, unless they are re-gated on path shape first. The in-crate ones (`keep`, `agents::running`, `agents::liveness`, `free::probe`) skip through `platform::skipped()`. `there_is_no_yes`, the malformed-list test and the fuzz half skip through `common::unsupported()`.
- `report::backups`' truncation and bit-flip tests get their binary property lists from `/usr/bin/plutil`, so they skip without it. To keep that coverage after the flip, commit the two binary plists as fixture bytes.

## Dependencies

- Stories "Move OS calls behind the platform module" and "Rule catalog split: shared and per-OS sections".
- Story "Build matrix: every target built, tested and packaged from one tag" (stage 2, and `release.yml` with its `enabled` list from stage 1).

## Attachments

-

---

*Last updated: 2026-10-28*

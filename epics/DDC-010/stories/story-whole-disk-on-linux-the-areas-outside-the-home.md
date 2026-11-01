---
type: story
id: "story-whole-disk-on-linux-the-areas-outside-the-home"
epic: "DDC-010"
title: "Whole disk on Linux: the areas outside the home"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [linux, disk, capability, research, cli]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Whole disk on Linux: the areas outside the home

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

On macOS, `ddc disk` answers "where did my disk go" beyond the home. It shows the volume's used and free space, the home row from the last scan, and a fixed table of areas (`disk/areas.rs`). Each area has a title, an explanation and what frees it. The table's paths are relative to `/System/Volumes/Data`, a macOS fact. On Linux the command refuses with exit 69, because Linux does not declare `whole-disk`.

A Linux disk fills in other places:
- system package caches (`/var/cache/apt`, `/var/cache/dnf`, `/var/cache/pacman/pkg`);
- the journal (`/var/log/journal`) and other logs;
- root Docker (`/var/lib/docker`, which the scan never walks);
- Snap (`/var/lib/snapd`, `/snap`) and system Flatpak (`/var/lib/flatpak`);
- the Nix store (`/nix/store`);
- old kernels and their modules (`/boot`, `/usr/lib/modules`);
- core dumps (`/var/lib/systemd/coredump`), a swap file, `/tmp` and `/var/tmp`;
- Linuxbrew's prefix (`/home/linuxbrew/.linuxbrew`);
- other users' homes;
- btrfs and ZFS snapshots.

Most of it is root-owned. An area the user cannot read is shown with an unknown size and the reason `needs-root`. It is never estimated.

**Rootless containers.** Story "Linux scanner and rule catalog" deferred root Docker's numbers to this capability. Rootless containers raise a second problem. In rootless Podman and Docker, files that belong to non-root users inside an image map to subordinate uids (100000 and up). Their folders are unreadable to the user, so the walk counts only those folders' own blocks under `vm.podman` and `vm.docker-rootless`. `du` cannot read them either, but the number is still wrong. `podman system df` and a rootless `docker system df` report the real size without root.

This story covers the reading half. Story "Freeing space outside the home on Linux: ddc disk free" covers the actions.

## Decomposition

- [ ] Research: a section "Outside the home" in [[research/where-developer-disk-space-goes-linux|Where a developer's disk space goes (Linux)]]. For each candidate area it gives:
  - the path on Debian and Ubuntu, Fedora and Arch;
  - a size;
  - whether an unprivileged user can measure it;
  - what frees it, and whether that needs root.
  
  Distro facts come from seeded containers or VMs. The owner's machines give sizes only, written without describing them. The section also decides whether other users' homes show as one row. Run the critic on the section.
- [ ] Areas per OS. `disk/areas.rs` gains a Linux table beside the macOS one, with paths relative to `/`, chosen by the platform module.
  - The `needs-root` reason.
  - The "other" row stays the volume's used space minus what was measured.
  - The Linuxbrew area is this story's.
- [ ] Measurement without root, where it exists, each with the timeouts and probe rules `disk/` uses today:
  - `journalctl --disk-usage`, which reports only the journals the user can read (groups `adm`, `systemd-journal`); say so in the area's explanation;
  - `docker system df --format json` for root Docker, when the daemon answers;
  - `podman system df` and the rootless `docker system df`;
  - `statvfs` per mounted filesystem in the table.
- [ ] The scan says "N folders unreadable (user namespace)" on `vm.podman` and `vm.docker-rootless` findings, and the whole-disk view shows the engine's own number beside them.
- [ ] Snapshots. When readable, show Snapper's, Timeshift's and ZFS's snapshots as an area. Decide whether Linux can declare `local-snapshots`, which lets a clean report say a snapshot still holds the freed space (`exec/mod.rs`). Declare it only if an unprivileged user can learn this reliably. Otherwise record why it stays absent.
- [ ] WSL. The Linux disk is a VHDX on the Windows drive, and Windows does not shrink it when files are deleted inside. Add a note to the volume row (`wsl --manage --set-sparse`, `Optimize-VHD`). It is not an action.
- [ ] `docs/disk.schema.json` and DDCKit: the new area ids and `needs-root`. The app is macOS-only, but it decodes every id without failing.
- [ ] Declare `whole-disk` on Linux.
  - Update ADR "Platform seam" §3.
  - The `disk` CLI tests leave `lacks(whole-disk)`.
  - The cases table runs `ddc disk` and `ddc disk --json` on Linux; its exit-69 expectation follows `CAPABILITIES` by itself.
  - The shared conda actions of `disk free` become available here. Their tests run on Linux.
- [ ] Tests:
  - a Linux fixture system root, beside the macOS `system-root` load tree;
  - the `needs-root` row;
  - the docker and podman probes with fake executables.

## Implementation Plan

## Acceptance Criteria

- [ ] The research section exists, with a size, a "measurable without root" answer and a freeing command per area, critic-approved.
- [ ] `ddc disk --json` on Linux shows the volume, the home row, and the Linux areas with sizes or `needs-root`, and validates against the disk schema (tests and cases).
- [ ] With a fake `podman`, the whole-disk view shows the engine's size, and the scan names the unreadable user-namespace folders under `vm.podman` (tests).
- [ ] `CAPABILITIES` for Linux holds `whole-disk`. The decision on `local-snapshots` is recorded in ADR "Platform seam" §3.

## Final Summary

## Technical Notes

- `docker system df` is the daemon's view. Never add it to a walked size.

## Dependencies

- Story "btrfs and ZFS on Linux: subvolumes, datasets, snapshots and stable identities", for the snapshots.

## Attachments

-

---

*Last updated: 2026-11-01*

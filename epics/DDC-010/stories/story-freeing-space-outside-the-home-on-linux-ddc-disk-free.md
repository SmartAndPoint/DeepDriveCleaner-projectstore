---
type: story
id: "story-freeing-space-outside-the-home-on-linux-ddc-disk-free"
epic: "DDC-010"
title: "Freeing space outside the home on Linux: ddc disk free"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [linux, disk, cli, safety]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Freeing space outside the home on Linux: ddc disk free

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

On macOS, `ddc disk free` runs the owning tool's own command for an area: conda environments and caches, simulator runtimes and brew. Each action is journaled. Once story "Whole disk on Linux: the areas outside the home" declares `whole-disk`, the shared conda actions run on Linux too. Everything else on a Linux disk is either Docker's or root's.

- **Docker.** Under the macOS rule, only native commands with an explicit target are safe: `docker image prune` and `docker builder prune`. Never `docker system prune` or `docker volume prune`, because a volume no container references can still hold a database. A member of the `docker` group runs these without root. The same holds for rootless Docker and for `podman image prune`. DDC-002's story "VM images: docker prune runs only while Docker Desktop answers" set the rule that they run only while the daemon answers.
- **Root-only areas.** Package caches, the journal, Snap revisions, system Flatpak runtimes, old kernels and the Nix store. `ddc` never runs `sudo` and never asks for a password. Each such area shows its command as text, for the user to run: `sudo apt clean`, `sudo journalctl --vacuum-size=…`, `sudo dnf clean packages`, `nix-collect-garbage`, `flatpak uninstall --unused`, `snap remove --revision …`.

## Decomposition

- [ ] Actions for `docker image prune`, `docker builder prune` and `podman image prune`, offered only while the engine answers, journaled, with the dry run listing what each would remove.
- [ ] Printed commands per root-only area, from the research section of the whole-disk story. They are never in the action list `ddc` runs.
- [ ] Record this split in the safety ADR, then run the critic on the amendment.
- [ ] A test that walks every Linux action's command line and refuses `sudo`, `pkexec`, `su` and `doas`.
- [ ] Cases: `ddc disk free --dry-run` on Linux.

## Implementation Plan

## Acceptance Criteria

- [ ] `ddc disk free --dry-run` on Linux lists the conda, docker and podman actions it would run, and the root-only commands as text it never runs (tests with fake executables and cases).
- [ ] A docker action runs only while the daemon answers, and is journaled (tests with a fake `docker`).
- [ ] No Linux action runs `sudo` or another privilege tool (test over every action's command line).
- [ ] The safety ADR records the split between run and printed commands, critic-approved.

## Final Summary

## Technical Notes

- `flatpak uninstall --unused --user` needs no root and acts on user installs only. The catalog story decides whether it is an action.

## Dependencies

- Story "Whole disk on Linux: the areas outside the home".

## Attachments

-

---

*Last updated: 2026-11-01*

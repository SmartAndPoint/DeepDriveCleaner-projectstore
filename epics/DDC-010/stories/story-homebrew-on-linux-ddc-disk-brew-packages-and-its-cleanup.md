---
type: story
id: "story-homebrew-on-linux-ddc-disk-brew-packages-and-its-cleanup"
epic: "DDC-010"
title: "Homebrew on Linux: ddc disk brew-packages and its cleanup"
status: planned
priority: p3
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [linux, homebrew, capability, cli]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Homebrew on Linux: ddc disk brew-packages and its cleanup

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | planned |
| **Priority** | p3 |
| **Assignee** | Evgenii Konev |

---

## Description

Homebrew runs on Linux. Its prefix is `/home/linuxbrew/.linuxbrew`, or `~/.linuxbrew` for a single-user install, and its cache is `~/.cache/Homebrew`. On Linux, `brew services` drives systemd user units.

On macOS, `ddc` uses Homebrew in three places:
- the `cache.homebrew` rule;
- `ddc disk brew-packages`, which lists formulae with their sizes, dependents and services and offers `brew uninstall` for orphans;
- `ddc disk free`'s brew actions (`brew cleanup`, `brew uninstall`, uninstalling orphans).

On Linux the capability `homebrew` is absent. The rule is macOS-only, and the commands refuse with exit 69.

## Decomposition

- [ ] Find the prefix from the running `brew` (`brew --prefix`, by the existing probe rules) or the two known prefixes. Never search the disk for it.
- [ ] `cache.homebrew` gains the Linux path `~/.cache/Homebrew`, with the same native command (`brew cleanup`) and probe (`brew --cache`).
- [ ] `ddc disk brew-packages` on Linux:
  - the same read commands (`brew info --json=v2 --installed`, `brew leaves`, `brew services list --json`);
  - services read from systemd through `brew services`, never through `systemctl` directly.
- [ ] `ddc disk free`'s brew actions on Linux, journaled as on macOS. The Linuxbrew area of the whole-disk view belongs to story "Whole disk on Linux: the areas outside the home".
- [ ] Declare `homebrew` on Linux, and update ADR "Platform seam" §3. The brew CLI tests that use a fake `brew` run on Linux. The cases runner derives the exit-69 expectation from `CAPABILITIES`, so `ddc disk brew-packages` runs on Linux once the capability is declared.
- [ ] A nightly CI job installs Homebrew on the Linux runner and runs `ddc disk brew-packages --json` against the real `brew`.

## Implementation Plan

## Acceptance Criteria

- [ ] `ddc scan` on Linux finds `~/.cache/Homebrew` under `cache.homebrew` (test).
- [ ] `ddc disk brew-packages --json` on Linux lists formulae, orphans and services through a fake `brew`, and validates against the disk schema (tests and cases).
- [ ] The brew `disk free` actions run and are journaled on Linux with a fake `brew` (tests).
- [ ] `CAPABILITIES` for Linux holds `homebrew`.

## Final Summary

## Technical Notes

- `/home/linuxbrew` is outside the user's home. The scan of `~` never walks it. Only the whole-disk area and `brew` itself see it.

## Dependencies

- None blocking. `disk free` with only `homebrew` declared is already allowed by `gate.rs`, so this story can land before or after the whole-disk stories.

## Attachments

-

---

*Last updated: 2026-11-01*

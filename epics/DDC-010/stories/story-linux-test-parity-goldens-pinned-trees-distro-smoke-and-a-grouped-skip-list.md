---
type: story
id: "story-linux-test-parity-goldens-pinned-trees-distro-smoke-and-a-grouped-skip-list"
epic: "DDC-010"
title: "Linux test parity: goldens, pinned trees, distro smoke and a grouped skip list"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [linux, tests, ci, gate]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Linux test parity: goldens, pinned trees, distro smoke and a grouped skip list

| Field | Value |
|---|---|
| **Epic** | [DDC-010](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

`.config/native-skips/linux.txt` lists the tests that skip on Linux (as of commit 2b95ac1). Some of them are about capabilities, and the other stories of this epic remove those. The rest exist because the tests were recorded or pinned on macOS:
- the goldens: `golden_deep`, `golden_deep_path_max`, `golden_links`, `golden_mixed`, `golden_names`, `golden_projects`, `golden_system_root`, `golden_users`, `golden_wide` and `golden_wide_root`;
- `a_change_to_one_rule_fails_the_golden_check`;
- the pinned load-tree hashes (`the_s_hashes_are_pinned`).

So a change that alters Linux scan output is caught only on macOS, by tests of the macOS catalog. Nothing pins the Linux output.

The release's musl archives are smoke-tested only on the Ubuntu runner. A static binary should run on every distro, but nothing checks that, or each distro's own facts.

## Decomposition

- [ ] Linux goldens.
  - `bench/golden/` gains a Linux set beside the macOS one: the same trees, built from the Linux catalog with Linux's `PATH_MAX`.
  - `make golden-record` on Linux writes it, run with `scripts/native-linux.sh --make golden-record`.
  - The golden tests compare against their own OS's set.
  - When a change alters both sets, record both in the same PR, under the goldens rule: the diff follows only from the PR's own change.
- [ ] Pinned load trees: `the_s_hashes_are_pinned` pins a Linux hash beside the macOS one.
- [ ] Distro smoke in `release.yml`, after the musl build. The archive runs in `debian:stable`, `fedora:latest`, `archlinux:latest` (x86_64 only) and `alpine:latest`, as a non-root user with a seeded home:
  - `--version`, `rules`, `scan --json` validated against the schema, `clean --dry-run`, `du`;
  - the suggested clean command, pasted back into `sh`.
- [ ] The skip list.
  - Group `.config/native-skips/linux.txt` by reason, with a comment on each group: capability (shrinks with this epic), macOS subject (Xcode, iOS backups, app bundles, `cp -c` clones, `diskutil`, `tmutil`, the macOS-only rules), and environment (needs the CI mounts, from story "Linux CI mounts: a filesystem matrix job with root-only steps").
  - `scripts/native-skips.sh` refuses a name outside a group.
- [ ] The local mounts step, moved here from story "Linux CI mounts: a filesystem matrix job with root-only steps". `scripts/native-linux.sh` passes a mounts folder into its container (`-v <base>`, `--env-file`), so the environment group can run on a Linux machine where loop mounts are allowed. That needs a version of `scripts/ci-mounts.sh` that undoes what it sets up (exports, Samba share, zpool), because today it refuses outside CI. The native-runs runbook then describes the step.

## Implementation Plan

## Acceptance Criteria

- [ ] The golden tests run on Linux against a Linux set and pass. A rule change that alters Linux output fails them on Linux (tests).
- [ ] `the_s_hashes_are_pinned` runs on Linux with a Linux hash.
- [ ] `release.yml` smoke-tests both musl archives on Debian, Fedora and Alpine, and the x86_64 archive on Arch.
- [ ] `.config/native-skips/linux.txt` is grouped by reason, and `scripts/native-skips.sh` refuses an ungrouped name (test of the script).

## Final Summary

## Technical Notes

- The goldens hold paths relative to the fixture root and no host facts. Recording them on Linux breaks no rule about the owner's machines.

## Dependencies

- Story "Linux CI mounts: a filesystem matrix job with root-only steps", for the environment group.

## Attachments

-

---

*Last updated: 2026-11-01*

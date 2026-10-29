---
type: story
id: "story-move-os-calls-behind-the-platform-module"
epic: "DDC-007"
title: "Move OS calls behind the platform module"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-29
external_refs: {}
tags: [rust, refactor, portability]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Move OS calls behind the platform module

| Field | Value |
|---|---|
| **Epic** | [DDC-007](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

This story does the move the seam ADR describes. Afterwards the workspace compiles for Linux and Windows, macOS behaves exactly as before, and the build matrix can run tests natively on all three systems. It is p1 because the build matrix's stage 2 waits on it.

OS-specific code today, found by `grep -rn 'libc::\|std::os::unix\|osascript\|launchctl'`:

| Where | What |
|---|---|
| `ddc-core/src/sys/*` | all FFI (bulk listing, fsops, procs, statfs, iopolicy, rlimit, signals, term) |
| `scan/walker.rs`, `scan/reader.rs` | `libc` errno mapping to `ReadErrorKind`, `std::os::unix` metadata |
| `report/backups.rs`, `report/contents.rs` | `libc::O_NOFOLLOW` in `backups.rs`; `libc::EACCES` in `contents.rs`, test module only |
| `report/after_clean.rs`, `rules/matcher.rs` | `std::os::unix` |
| `exec/system.rs`, `exec/delete.rs`, `exec/guard.rs` | `SIGKILL` of process groups, `std::os::unix` |
| `disk/mod.rs`, `disk/visitor.rs`, `disk/apfs.rs`, `disk/*` | macOS areas, APFS |
| `env.rs` | `HOME`, `~/Library` folders |
| `ddc-cli/src/launchd.rs`, `notify.rs`, `cmd/schedule.rs` | launchd, `osascript`, `std::os::unix` |
| `ddc-cli/src/main.rs`, `progress.rs`, `cmd/{diag,clean,clean_items,background,disk,schedule}.rs` | direct use of `ddc_core::sys` (iopolicy, rlimit, procs, signals, statfs, term) |
| `ddc-loadgen/src/{isolate,build,run,measure,target,verify}.rs` | direct use of `ddc_core::sys` (procs, usage, statfs, rlimit); the property tests of DDC-005's story "Coverage floor and property tests for the safety invariants" stand on it |
| `ddc-core/tests/common/`, `in_use.rs`, `clean_safety.rs`, `user_data.rs` and the other core test files (7 in all), and 6 CLI test files | `std::os::unix` in the test harness |

## Decomposition

- [ ] `ddc-core/src/platform/{mod,macos,linux,windows}.rs` (or folders). `sys/` moves under `platform::macos`. Linux and Windows get stubs that compile and return `Unsupported` for each capability, with `SUPPORTED = false`.
- [ ] Every row of the table above moves behind `platform`, or behind `cli::platform` for `ddc-cli`. Shared code calls only the seam.
- [ ] The test harness: helpers that need Unix move behind `cfg(unix)` helpers in `tests/common`. Tests that are macOS-only by nature are marked `cfg(target_os = "macos")` with a one-line reason. Platform-neutral tests (rules, plan, report on fixtures, property tests) carry no `cfg`.
- [ ] `make tools` adds the targets (`rustup target add x86_64-unknown-linux-gnu x86_64-unknown-linux-musl x86_64-pc-windows-msvc`). The gate's full tier runs `cargo check --workspace --all-targets --target <t>` for each.
- [ ] Before and after: `make golden-check` (DDC-005 story "Synthetic load trees: large, wide, deep and hostile"), plus the full macOS test suite.

## Implementation Plan

## Acceptance Criteria

- [ ] `cargo check --workspace --all-targets` passes for `x86_64-unknown-linux-gnu`, `x86_64-unknown-linux-musl` and `x86_64-pc-windows-msvc`, in the gate.
- [ ] On macOS the full test suite passes, and the golden outputs are identical except for the fields the seam ADR added.
- [ ] No file under `src/` outside `platform/` (in `ddc-core`) and `cli/platform` (in `ddc-cli`) references `libc::`, `std::os::unix`, `std::os::windows` or `cfg(target_os)`, and no file in `ddc-cli` outside `cli/platform`, nor in `ddc-loadgen`, references `ddc_core::sys::` or `ddc_core::platform::macos::`, enforced by a gate check (a grep, or a `ddc-shape` layer rule once DDC-006 has it).
- [ ] `unsafe_code` is denied everywhere except `platform/`.
- [ ] `projectstore:reviewer` confirmed the move preserved behaviour.

## Final Summary

## Technical Notes

- Do this before DDC-006's story "Burn down the baseline: split long files and functions". Both touch `scan/walker.rs`, `exec/*` and `disk/*`, so splitting first would mean doing the work twice.
- `sys/` becomes `platform/macos/`. DDC-006's story "Clippy at full strength: curated lints with reasons" names `sys/` (`undocumented_unsafe_blocks`, `#![allow(unsafe_code)]`); whichever lands second uses the new path.
- Prefer `rustix` (safe syscall wrappers, already in `Cargo.lock` transitively) for Linux, and `windows-sys` (also in the lock file) for Windows.

## Dependencies

- Story "Platform seam: ADR and capability model".
- DDC-005 stories "Mandatory gate before every commit" and "Coverage floor and property tests for the safety invariants" (safety net).

## Attachments

-

---

*Last updated: 2026-10-29*

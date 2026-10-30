---
type: story
id: "story-clippy-at-full-strength-curated-lints-with-reasons"
epic: "DDC-006"
title: "Clippy at full strength: curated lints with reasons"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [rust, clippy, lints, conventions]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Clippy at full strength: curated lints with reasons

| Field | Value |
|---|---|
| **Epic** | [DDC-006](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

Clippy runs at its default level today. `pedantic` + `nursery` report 399 distinct warnings in `src/` and 462 with tests (2026-10-28). Turning whole groups on would bury the useful lints under style noise. Leaving them off misses real problems: 43 cast warnings in `src/`, 7 `needless_pass_by_value`, 7 functions over 100 lines.

This story picks each lint deliberately, sets the thresholds the ADR decided, fixes the warnings, and requires a reason on every exception.

## Decomposition

- [ ] `pedantic` as a group at `warn`, with `priority = -1` (otherwise `lint_groups_priority` fails the build). The gate's `-D warnings` denies it. Turn off by name, with a comment, only the lints the ADR rejects (candidates: `must_use_candidate`, `missing_errors_doc` for crate-internal APIs, `module_name_repetitions`).
- [ ] From `nursery`, only the chosen lints (candidates: `use_self`, `redundant_clone`, `option_if_let_else`, `cognitive_complexity`; `missing_const_for_fn` only if the ADR wants it).
- [ ] From `restriction`:
  - `allow_attributes_without_reason`;
  - `unwrap_used` and `expect_used` (with `allow-unwrap-in-tests` and `allow-expect-in-tests`), only if the ADR adopts them after the design story's count (about 114 sites in `src/` today, outside the 399). If adopted, each becomes `?`, a checked match, or `#[expect(…, reason)]`;
  - `undocumented_unsafe_blocks` for `platform/macos/` and `platform/op_stats.rs` (DDC-007's story "Move OS calls behind the platform module" moved `sys/` there).
  
  No `as_conversions`: it fires on every `as`, lossless or not, and the pedantic `cast_possible_truncation`, `cast_sign_loss` and `cast_possible_wrap` already catch the lossy ones.
  
  `print_stdout` and `print_stderr` stay as `#![deny(…)]` in `ddc-core/src/lib.rs`. A crate cannot add lints on top of `[lints] workspace = true`, and the workspace table cannot be crate-specific. `allow_attributes` (which forces `#[expect]`) only if the ADR accepts the `tests/common` exception below.
- [ ] Thresholds in `clippy.toml`, from the ADR: `too-many-lines-threshold`, `cognitive-complexity-threshold`, `too-many-arguments-threshold`, `type-complexity-threshold`, and `excessive-nesting-threshold` (hard limit + 2, see the design story).
- [ ] Fix the warnings crate by crate, one lint family per commit, so each commit stays reviewable and behaviour-preserving. Casts get checked conversions (`try_from`) or an `#[expect]` with the reason the value fits.
- [ ] The 17 `#[allow]` sites today, 7 of them in `src/`:
  - the `disallowed_methods` ones in `src/` (`env.rs`, `store.rs`, `exec/delete.rs`, `launchd.rs`) become `#[expect(…, reason = "…")]`;
  - `cmd/schedule.rs` `install_with` carries `#[allow(clippy::too_many_arguments)]` (8 parameters). It is also a shape violation (hard limit 6): it goes to the burn-down list, or gets a `shape.toml` exception with a reason;
  - the 8 `#[allow(clippy::disallowed_methods)]` in `tests/` (the `DDC_UPDATE_SCHEMA` reads) become `#[expect(…, reason = "…")]`;
  - `#![allow(unsafe_code)]` in `platform/macos/mod.rs` and `platform/op_stats.rs` gains a reason;
  - `tests/common`'s `#![allow(dead_code)]` stays an `allow` with a reason: `common/` is compiled into every test binary, and `#[expect]` would fail in whichever binary uses all of it.
- [ ] Functions over the hard length limit go to the burn-down story's list. They do not get an `#[expect]` here.

## Implementation Plan

## Acceptance Criteria

- [ ] `make gate` runs clippy with the curated set and passes.
- [ ] `grep -rnE '#!?\[(allow|expect)\(' crates` lists only attributes with a `reason`, enforced by `allow_attributes_without_reason`.
- [ ] Every `unsafe` block in `platform/macos/` and `platform/op_stats.rs` has a `// SAFETY:` comment, enforced by `undocumented_unsafe_blocks`.
- [ ] All tests pass. In tests, lint-only edits are allowed; no assertion, fixture or expected value changed (reviewer checks the diff).

## Final Summary

## Technical Notes

- Clippy lints change between toolchains. The pinned toolchain in `rust-toolchain.toml` keeps the set stable. A toolchain bump is a separate commit that fixes new findings.
- Run clippy with `PATH=$HOME/.cargo/bin:$PATH` (as the Makefile does). Homebrew's `cargo-clippy` 1.81 shadows it on the reference machine and reports nothing.

## Dependencies

- Story "Code shape conventions and the checker's design" (thresholds and the accepted lint list).
- DDC-005 story "Mandatory gate before every commit".

## Attachments

-

---

*Last updated: 2026-10-28*

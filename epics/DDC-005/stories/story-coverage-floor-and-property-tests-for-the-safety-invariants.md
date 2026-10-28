---
type: story
id: "story-coverage-floor-and-property-tests-for-the-safety-invariants"
epic: "DDC-005"
title: "Coverage floor and property tests for the safety invariants"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [rust, testing, coverage, property-tests, fuzzing]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Coverage floor and property tests for the safety invariants

| Field | Value |
|---|---|
| **Epic** | [DDC-005](../epic.md) |
| **Status** | planned |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The ~315 tests check examples: a given tree gives a given finding. Nobody knows which lines no test reaches. The safety promises hold for any tree, not just the examples, and for those promises examples are the weakest kind of evidence.

The promises come from the safety ADR and DDC-001:
- no protected folder, `user-data` item or VM directory is ever planned for deletion;
- sizes are counted once;
- the plan does not depend on walk order;
- an interrupted clean leaves a consistent state.

This story measures coverage and sets a floor that only rises. It states the invariants as property tests over generated trees, checks the rule matcher against a simple reference implementation, tests interruption, and fuzzes the parsers that read outside input.

## Decomposition

- [ ] Add `llvm-tools-preview` to `rust-toolchain.toml`. `cargo llvm-cov` over the workspace gives line and region coverage per crate and per module, as text and HTML. `make coverage` writes the report to `target/coverage/`. It runs in CI on every push, not in the local hooks.
- [ ] Record today's numbers. The floor per crate is the measured value minus 1 point, in `coverage-floor.toml`. A CI check fails when coverage drops below the floor. The floor is raised (ratchet) only by `make coverage-ratchet`, run by the owner on `main` from the latest CI numbers, never from a branch or by CI itself.
- [ ] Target for the safety path: line coverage ≥ 90% for `exec/mod.rs`, `exec/delete.rs`, `exec/guard.rs`, `exec/preflight.rs`, `exec/journal.rs`, `plan.rs` and `rules/classify.rs`. `sys/` and the CLI's rendering are adapters, excluded from the target and covered by integration tests. Add tests until the target is met, or record why a line is unreachable.
- [ ] `proptest` strategies for small random trees: projects, caches, protected folders, user data, keep markers, nested claims, hard links, symlinks.
- [ ] Property: for any tree, no plan contains a path inside a protected folder, a `user-data` item without explicit selection, or a VM directory.
- [ ] Property: the report's totals equal the sum over distinct findings. Nested claims and hard links are counted once.
- [ ] Property: findings and plan are the same for 1 thread and N threads, and for any order of directory entries.
- [ ] Differential test: the rule matcher agrees with a deliberately simple reference matcher (linear scan over all rules) on generated paths.
- [ ] Property: every JSON report round-trips and validates against `docs/scan.schema.json` (and the clean, progress and disk schemas). `generated_at` and `duration_ms` are excluded from equality.
- [ ] Interruption tests: `kill -TERM` during a scan of an S tree and during a clean of 10⁴ items of a generated tree. `clean` ends by the signal it received (it re-raises it, `clean_items.rs`); `scan` has no handler and simply dies. In both cases the lock is released (kernel `flock`), `last-scan.json` is either the previous or the complete new file (atomic write in `store.rs`), a second run works normally, and nothing outside the plan was removed.
- [ ] Fuzz targets (`cargo fuzz`, nightly, in the nightly workflow): `disk/plist.rs`, reading `settings.json` and `last-scan.json`, parsing the `--items` list. `make fuzz` runs each for a set time; crashes become regression tests.

## Implementation Plan

## Acceptance Criteria

- [ ] `make coverage` produces per-crate and per-module numbers. The floor file exists, and a drop below it fails CI.
- [ ] The named safety modules reach ≥ 90% line coverage, or each uncovered line has a recorded reason.
- [ ] The three invariant properties and the matcher differential test run in the fast gate with ≥ 256 cases each. A seeded bug (e.g. a protected-folder check removed) is caught by a property test. This is shown once, on a throwaway branch.
- [ ] The interruption tests pass in the full gate.
- [ ] Each fuzz target ran for ≥ 10 minutes without a crash, or its crash is fixed and kept as a test.

## Final Summary

## Technical Notes

- Property and interruption tests write to temporary directories only, with `HOME` and `--data-dir` pointed at scratch folders (the isolation rules of story "Memory and CPU under load: peak memory, leaks and CPU time"). Proptest's shrinking keeps failures small enough to read.
- The FFI in `sys/` cannot run under Miri. Its safety rests on these tests, the leak check and the nightly AddressSanitizer run in story "Memory and CPU under load: peak memory, leaks and CPU time".

## Dependencies

- Story "Mandatory gate before every commit" (the gate this story adds to).
- Story "Synthetic load trees: large, wide, deep and hostile" for the interruption tests' trees.

## Attachments

-

---

*Last updated: 2026-10-28*

---
type: epic
id: "DDC-005"
title: "Production-grade Rust core: mandatory gate and load testing"
status: in-progress
priority: p1
created: 2026-10-28
updated: 2026-10-29
external_refs: {}
tags: [rust, quality, testing, performance, ci]
code_refs: [".config/nextest.toml", ".config/tools.mk", ".githooks/pre-commit", ".githooks/pre-push", ".github/workflows/gate.yml", ".github/workflows/nightly.yml", "Makefile", "deny.toml", "scripts/ci-green.sh", "crates/ddc-core/Cargo.toml", "Cargo.toml", "crates/ddc-core/src/sys/procs.rs", "crates/ddc-loadgen/**", "bench/golden/**", "crates/ddc-cli/Cargo.toml", "crates/ddc-cli/tests/golden_cli.rs", "scripts/load-image.sh", "bench/compare_load.py", "bench/test_compare_load.py", "bench/dhat_top.py", "bench/test_dhat_top.py", "bench/load-limits.tsv", "crates/ddc-core/src/heap.rs", "crates/ddc-core/src/sys/usage.rs", "crates/ddc-core/src/sys/mod.rs", "crates/ddc-core/src/exec/system.rs", "crates/ddc-cli/src/main.rs", "crates/ddc-cli/src/cli.rs", "crates/ddc-cli/src/cmd/background.rs", "crates/ddc-cli/src/cmd/clean_items.rs", "crates/ddc-cli/tests/load_cli.rs", "crates/ddc-cli/src/launchd.rs", "crates/ddc-cli/src/progress.rs", "crates/ddc-cli/tests/clean_items_cli.rs", "crates/ddc-core/src/lib.rs", "crates/ddc-core/tests/scanner.rs", "crates/ddc-core/src/stats.rs", "crates/ddc-core/src/sys/op_stats.rs", "crates/ddc-core/src/sys/bulk.rs", "crates/ddc-core/src/scan/reader.rs", "crates/ddc-core/src/report/contents.rs", "crates/ddc-core/src/report/mod.rs", "crates/ddc-core/src/rules/classify.rs", "crates/ddc-core/src/rules/matcher.rs", "crates/ddc-cli/tests/structure.rs", "scripts/build-dist.sh", "scripts/build-app.sh", "crates/ddc-core/src/agents/claude.rs", "crates/ddc-core/src/agents/codex.rs", "crates/ddc-core/src/agents/mod.rs", "crates/ddc-core/src/disk/mod.rs", "crates/ddc-core/src/exec/delete.rs", "crates/ddc-core/src/report/project.rs", "crates/ddc-core/src/rules/toolchain.rs", "crates/ddc-core/src/exec/preflight.rs", "crates/ddc-core/src/rules/mod.rs", "crates/ddc-loadgen/src/verify.rs", "crates/ddc-loadgen/src/scaling.rs", "crates/ddc-loadgen/src/limits.rs", "crates/ddc-core/src/plan.rs", "crates/ddc-core/src/report/after_clean.rs", "crates/ddc-core/src/exec/report.rs", "crates/ddc-core/src/exec/mod.rs", "crates/ddc-loadgen/src/main.rs", "crates/ddc-loadgen/src/lib.rs", "crates/ddc-loadgen/tests/profiles.rs", "crates/ddc-loadgen/Cargo.toml", "crates/ddc-loadgen/src/isolate.rs", "crates/ddc-loadgen/src/measure.rs", "clippy.toml", "crates/ddc-cli/tests/scan_cli.rs", "crates/ddc-core/src/exec/guard.rs", "crates/ddc-core/src/rules/visitor.rs", "bench/coverage_floor.py", "bench/test_coverage_floor.py", "bench/coverage-floor.tsv", "crates/ddc-core/tests/matcher_reference.rs", "crates/ddc-core/src/report/volumes.rs", "crates/ddc-core/tests/common/fakes.rs", "crates/ddc-loadgen/tests/properties.rs", "crates/ddc-loadgen/tests/order.rs", "crates/ddc-cli/tests/contract/mod.rs", "crates/ddc-cli/tests/interrupt_cli.rs", "crates/ddc-core/tests/brew_properties.rs", "crates/ddc-core/tests/simctl_properties.rs", "crates/ddc-core/tests/clean_safety.rs", "crates/ddc-core/src/store.rs", "fuzz/Cargo.toml", "fuzz/Cargo.lock", "fuzz/fuzz_targets/disk_plist.rs", "fuzz/fuzz_targets/listed_items.rs", "fuzz/fuzz_targets/action_list.rs", "fuzz/fuzz_targets/settings.rs", "fuzz/fuzz_targets/last_scan.rs", "crates/ddc-core/tests/fuzz_regressions.rs", "crates/ddc-core/src/disk/plist.rs", "crates/ddc-core/src/settings.rs", "fuzz/.gitignore", "fuzz/regressions/**", "crates/ddc-cli/tests/background_cli.rs", "crates/ddc-cli/tests/brew_packages_cli.rs", "crates/ddc-cli/tests/clean_cli.rs", "crates/ddc-cli/tests/diag_cli.rs", "crates/ddc-cli/tests/disclaimer_cli.rs", "crates/ddc-cli/tests/disk_agents_cli.rs", "crates/ddc-cli/tests/disk_cli.rs", "crates/ddc-cli/tests/disk_free_cli.rs", "crates/ddc-cli/tests/du_cli.rs", "crates/ddc-cli/tests/notify_cli.rs", "crates/ddc-cli/tests/readme_cli.rs", "crates/ddc-cli/tests/rules_cli.rs", "crates/ddc-cli/tests/settings_cli.rs"]
review_status: reviewed
reviewed_at: 2026-10-28
---

# DDC-005: Production-grade Rust core: mandatory gate and load testing

| Field | Value |
|---|---|
| **Status** | in-progress |
| **Priority** | p1 |
| **Created** | 2026-10-28 |
| **Updated** | 2026-10-29 |

---

## Goal

No change reaches `main` without one mandatory check. The check formats, lints, tests, checks dependencies and builds the docs. The core also gets a load test suite. It turns memory growth, CPU time and super-linear algorithms into numbers with limits, so a regression fails a test before a user's disk finds it.

Success criteria:

- A local git hook refuses a commit with a format error, a clippy warning or a failing test. The hook is versioned in the repository. CI re-runs the gate on every push; a red run on `main` is reported, and `make dist` refuses to build a commit whose CI run is not green.
- Re-applying the per-finding directory read that commit `aa64c83` removed (step 2) makes a test in the gate fail, on any machine and without a wall-clock limit. Its per-claim scan (step 1) fails the gate when its comparison is counted, as the runbook requires; restored verbatim, an uncounted comparison loop moves no counter, and whether the CPU slope of `make scaling` at L shows it is measured once and recorded in story "Complexity guard: operation counts and scaling exponents".
- On generated trees of 10⁴ to 10⁶ entries, scan and report time and peak memory grow with a log-log slope ≤ 1.2. macOS `leaks --atExit` reports no leaked allocation after a scan.
- No `make load` run reads or writes the owner's real home, data directory, settings, LaunchAgents, notifications or Trash. (The owner's own `make bench-real` reads the home by design, read-only.)
- Every number is written to a dated file in `bench/`, and a script compares it with the previous run.

## Context

On 2026-10-28 commit `aa64c83` ("scan large system trees without quadratic steps") fixed two steps in report building (`report/mod.rs`):
1. each claim was looked up by a scan over all finding paths;
2. each finding re-read its project folder's activity (`report/project.rs` `last_activity`, a `read_dir` of the project root).

Both were invisible on the small temporary trees the tests use. They surfaced only on large real system trees, with many sibling folders such as per-user folders. Step 2 is a repeated directory read, not a lookup: catching it takes a counter of directory entries read, not just of lookups.

What exists today:

- About 315 tests (`#[test]`, working tree of 2026-10-28), all functional, on small temporary trees. No test touches the real `$HOME`.
- `make check` runs `cargo fmt --check`, `cargo clippy … -- -D warnings` and `cargo test`. `make dist` runs it before a release. No git hook and no CI enforce it: there is no `.github/` folder.
- `bench/` holds the DDC-001 baseline of the reference machine (`check_baseline.py`). It measures coverage of the findings, not time or memory.
- `unsafe` code exists only in `ddc-core/src/sys/` (FFI to `getattrlistbulk`, `libproc`, `statfs`, signals and others). The rest of the workspace denies it.
- `rust-version = "1.89"` is declared, but only toolchain 1.99.1 is ever built.
- Every process is short-lived. The app spawns the CLI per scan or clean. Here "memory leak" means memory that grows faster than the input, and allocations still live at exit. It does not mean a daemon growing over days.
- A `--background` scan runs with 3 threads. When scheduled by `ddc schedule`, its low priority comes from that LaunchAgent plist (`LowPriorityIO`, `Nice 10`), not from code. The app's own scheduler plist sets neither: a DDC-002 follow-up, outside this epic. A scan with `--background` also posts a notification unless `--no-notify` is given.

Where a check runs:
- Only machine-independent assertions run in CI and the gate: operation-count ratios, manifests, leaks, schemas.
- Absolute limits (seconds, bytes per entry, CPU per entry) are checked on the reference machine only. They are recorded and compared, and a regression is reported, not used to fail CI.

Decisions the epic relies on:
- [[adr/rust-core-library-and-cli-with-json-output|Rust core: library + CLI with JSON output]]
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]. Load tests delete only inside generated trees.

## Stories

| Story | Status | Description |
|-------|--------|-------------|
| [story-mandatory-gate-before-every-commit](./stories/story-mandatory-gate-before-every-commit.md) | done | `make gate-fast` / `make gate`: fmt, clippy, tests with timeouts, docs, cargo-deny, unused dependencies, MSRV. Versioned pre-commit and pre-push hooks, a GitHub Actions workflow on every push and a nightly one |
| [story-coverage-floor-and-property-tests-for-the-safety-invariants](./stories/story-coverage-floor-and-property-tests-for-the-safety-invariants.md) | planned | Line coverage measured and ratcheted from `main`. Property and differential tests for the safety and counting invariants. Interruption tests. Fuzz targets for the parsers |
| [story-synthetic-load-trees-large-wide-deep-and-hostile](./stories/story-synthetic-load-trees-large-wide-deep-and-hostile.md) | done | A deterministic generator of 10⁴–10⁷-entry trees on a disposable disk image, with a manifest of the expected findings and plan. It refuses any target that is not its own image or a marked temporary folder (S only on macOS) |
| [story-complexity-guard-operation-counts-and-scaling-exponents](./stories/story-complexity-guard-operation-counts-and-scaling-exponents.md) | in-progress | Cost counters (directory entries read, claim lookups, rule matches, allocations) under a feature, asserted linear in the gate. Scaling slopes on larger trees. Regression tests that reproduce both steps of `aa64c83` |
| [story-memory-and-cpu-under-load-peak-memory-leaks-and-cpu-time](./stories/story-memory-and-cpu-under-load-peak-memory-leaks-and-cpu-time.md) | in-progress | An isolated load harness. Peak RSS per entry, `leaks --atExit`, AddressSanitizer for `sys/`, heap profiles, CPU time, idle waits. Dated results, a comparison script and a profiling runbook |
| [story-load-run-on-the-reference-machine-including-system-folders](./stories/story-load-run-on-the-reference-machine-including-system-folders.md) | planned | `make bench-real`: read-only release-build scans of the home, `~/Projects` and the whole-disk areas, run by the owner, recorded and compared. Advisory, not a release gate |
| [story-clean-of-listed-items-look-each-item-up-by-path-not-among-all-findings](./stories/story-clean-of-listed-items-look-each-item-up-by-path-not-among-all-findings.md) | planned | `clean --items` looks each listed item up among all findings (O(items × findings): `clean-items` user-time fit 1.82 at L); index the findings by path, keep the safety model's refusals, judge `clean-items` in `make scaling`. p2. |

Implementation order: the gate, then the trees, then memory and CPU (the harness and the runbook), then the complexity guard, then the reference-machine run. The coverage story can start after the gate; its interruption tests wait for the trees.

## Expected Results

- [x] `make gate-fast` and `make gate` are the mandatory checks. The git hooks and CI run them, and their parts and timings are documented.
- [ ] Coverage per crate is measured. The floor only rises. The safety invariants hold under property tests.
- [ ] `make load` generates trees, runs the complexity, memory and CPU checks in isolation, and writes `bench/load-<date>-<size>.tsv`.
- [ ] A reintroduced quadratic step fails the gate: step 2 of `aa64c83`, or step 1 with its comparison counted.
- [ ] A runbook explains how to profile the core (flamegraph, heap profile, `leaks`, counters) when a number regresses.

## Dependencies

- GitHub Actions on the public repository (macOS runners are free for public repositories).
- Pinned tool versions: cargo-nextest, cargo-deny, cargo-llvm-cov (plus the `llvm-tools-preview` component), cargo-machete, proptest, dhat. A nightly toolchain only for fuzzing and AddressSanitizer, in the nightly workflow, outside the gate.
- Space for a sparse disk image of up to 10⁷ empty files, created and removed by the generator.

## Open Questions

- [x] *Measured 2026-10-28, owner to confirm:* pre-commit runs the whole fast tier. That is 42 s warm, and 78 s after a core change: over the 60 s proposal, set by four slow `recheck` tests (5 s alone, 35 s under load). Pre-push takes 85–97 s warm and 170 s cold. If 78 s is too slow, speed those tests up first, before dropping integration tests from pre-commit. Original question: what gate time is acceptable, given agents commit many times a day? Proposal: pre-commit ≤ 60 s warm (fmt, clippy, unit and integration tests, S-size load checks), or, if that proves too slow, fmt + clippy + unit tests only, with integration and S load moved to pre-push; pre-push ≤ 5 min (adds docs, deny, machete, MSRV, leaks); coverage and M-size load in CI on every push; L size, fuzzing and AddressSanitizer nightly. Set after measuring in the gate story.
- [x] Should CI also run the app's `swift test`? Decided: yes, as the `app` job in `gate.yml`, after building the debug CLI so the real-CLI tests do not skip themselves. It is outside this epic's core scope.
- [x] Is the MSRV claim worth keeping? Decided: keep 1.89. It is the real floor: `File::try_lock` in `store.rs` needs 1.89. `make gate` checks it with `cargo +1.89 check` (`make msrv`), and that check passed on 2026-10-28. It matters for DDC-007's `cargo install` and Linux packagers.

## Related

- [[epics/DDC-001/epic|DDC-001 · Disk space cleaner CLI for developers]]: the core this epic hardens; its success criteria (≤ 60 s home scan) stay.
- [[epics/DDC-002/epic|DDC-002 · Menu bar app]]: its story "Whole disk: what lies outside the home folder" adds `ddc disk`, which the load matrix covers.
- [[epics/DDC-006/epic|DDC-006 · Code shape gate: measurable conventions that guide humans and agents]]: adds its checker to this gate; its refactors rely on this epic's tests and golden outputs.
- [[epics/DDC-007/epic|DDC-007 · Standalone CLI on macOS, Linux and Windows: portable core, release builds and installers]]: its build matrix extends this gate's workflow; this epic's tests are the safety net for its move of OS calls behind the platform module.

---

*Last updated: 2026-10-29*

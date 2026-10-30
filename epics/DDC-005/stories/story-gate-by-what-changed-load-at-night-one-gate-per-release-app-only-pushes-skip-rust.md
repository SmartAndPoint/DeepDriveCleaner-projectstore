---
type: story
id: "story-gate-by-what-changed-load-at-night-one-gate-per-release-app-only-pushes-skip-rust"
epic: "DDC-005"
title: "Gate by what changed: load at night, one gate per release, app-only pushes skip Rust"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-30
updated: 2026-10-30
external_refs: {}
tags: [ci, quality, git-hooks, release]
code_refs: [".config/rust-free-paths", "scripts/gate-scope.sh", "scripts/tests/test_gate_scope.py", ".githooks/pre-commit", ".githooks/pre-push", ".github/workflows/gate.yml", ".github/workflows/nightly.yml", ".github/workflows/release.yml", "Makefile", "scripts/ci-green.sh", "scripts/tag-release.sh", "scripts/require-tag.sh", "fuzz/Cargo.lock", "crates/ddc-core/tests/matcher_reference.rs"]
specs: []
started_at: 2026-10-30
closed_at: "2026-10-30T11:35:43.001Z"
plan_updated_at: 2026-10-30
---

# Gate by what changed: load at night, one gate per release, app-only pushes skip Rust

| Field | Value |
|---|---|
| **Epic** | [DDC-005](../epic.md) |
| **Status** | done |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

The tiers from story "Mandatory gate before every commit" (DDC-005) are right, but on 2026-10-30 they cost too much for every change. The owner said that new features would take a very long time to reach production, especially features that do not touch the core, such as GUI work.

Measured on 2026-10-30, as the average of the last 5 green Gate runs on `main`:

| Gate job | Time |
|---|---|
| `load` (M-size trees, every profile) | 16.5 min (15–18) |
| `gate` (`make gate`) | 6.8 min |
| `coverage` | 3.6 min |
| `app` (`make app-test`) | 2.0 min |
| `release-build` | 1.5 min |
| `check` Linux / Windows | 0.4 / 1.2 min |

The run takes about 17 minutes of wall time, and `load` alone sets it. `make tag` and `make dist` wait for the run (`scripts/ci-green.sh`).

A release then runs `make gate` four times on the same commit:
1. in the pre-push hook of the branch;
2. in CI;
3. in the pre-push hook of the tag push in `scripts/tag-release.sh`;
4. in `make dist`, which also repeats `make app-test` from CI's `app` job.

A change confined to the Swift app, the website or the release notes runs the whole Rust gate locally and in CI, though nothing it touches can change a Rust result.

Separately, Nightly's `fuzz` job has failed since 0.8.0. `fuzz/Cargo.lock` still pins `ddc-core 0.7.0`, so `cargo metadata --locked` refuses it. Every version bump makes that lock stale, and nothing in the gate reads it.

This story keeps every check, and moves each one to where it pays:

1. **M-size load at night.** The `load` job moves from `gate.yml` to `nightly.yml`, next to the L-size one.
   - At M, with `LOAD_LIMITS=record`, it does not judge performance limits. It does still judge correctness: each tree against its manifest, and the release CLI's exit and the items left after a clean (`scripts/load-image.sh`, `ddc-loadgen run`).
   - That correctness check now runs on `main` every night instead of on every push. A fault that shows only at 10^5 entries is found within 24 hours, not before the push.
   - The S-size trees stay in `make gate`. The owner chose this trade on 2026-10-30.
2. **One gate per release.**
   - `make dist` drops its own `gate` and `app-test`.
   - `ci-green` for `make tag` and `make dist` requires a run on `HEAD` itself in which the `gate`, `coverage` and `release-build` jobs all succeeded. That is the same commit, the same `make gate` and the same `make app-test`.
   - The tag push skips the pre-push hook's gate. A tag adds no branch, `gate.yml` ignores tags, and a release checks its commit again (`ci-green --full`, `require-tag.sh`, and `release.yml`'s `ci-green --full`). A tag pushed by hand on an unpushed commit is ungated, but it cannot be released.
3. **Gate by what changed.** A commit or push that touches only paths on one list skips the Rust tiers in the pre-commit hook, the pre-push hook and CI's Rust jobs.
   - The list is held in one place, read by the hooks and by CI, and guarded by a test.
   - It holds the app's own sources, the website and the release notes.
   - Any doubt runs the full gate: an unknown base, a failed `gh` call, an empty change set, or a path off the list.
4. **Fuzz lock checked.**
   - `fuzz/Cargo.lock` is refreshed to 0.8.0.
   - `make gate` checks it with `--locked` on the stable toolchain, next to `deny` and `machete`, so it fails in seconds.
   - The release runbook names the refresh as part of the version bump.

The guarantee `make dist` relies on does not depend on item 3. A release needs a full green run on the released commit itself. A fault in the path list can only make a development push wrongly green. Nightly's `make gate` on `main` catches that within a day, and `make tag` refuses it.

## Decomposition

- [x] `.config/rust-free-paths` holds the list.
  - Entries:
    - `app/`, except `app/Packages/DDCKit/Tests/DDCKitTests/Fixtures/` (read by `crates/ddc-core/tests/disk.rs`) and `app/DeepDriveCleaner.xcodeproj/project.pbxproj` (read by `packaging/install/test_install.py`);
    - `website/`;
    - `docs/releases/`.
  - Not on the list: the rest of `docs/` (Rust tests read `docs/*.schema.json`), `packaging/` (`make install-test` is in `make gate`), and the root documents (`readme_cli.rs`, `test_install.py`).
  - Semantics:
    - Paths are repo-relative and matched at the root.
    - An entry ending in `/` is a folder prefix; any other entry is one file.
    - `!` takes a path back off the list.
    - The list file itself is off the list.
- [x] `scripts/gate-scope.sh` is the one classifier. It prints `rust` or `skip`, and in `push` and `paths` mode also `swift`; `staged` and `ci` fold `swift` into `skip`. It exits 0, and a usage error prints `rust` and exits 2. Callers treat anything other than `skip` or `swift` as `rust`, empty output included. Modes:
  - `paths`, newline-separated on stdin, for the guard test.
  - `staged`, for pre-commit. An empty staged set answers `rust`.
  - `push`, which reads git's pre-push lines on stdin:
    - deletions and `refs/tags/*` lines are ignored, and a push of nothing else answers `skip`;
    - a branch line beside a tag is judged as usual;
    - a new branch is diffed against its merge base with `origin/main`;
    - an existing ref is diffed against its remote sha, which must exist locally;
    - empty stdin, or nothing left to judge, answers `rust`;
    - `swift` means the Rust tiers are skipped but `app/Packages/` changed.
  - `ci <sha>`, for the Gate's `scope` job (see below).
  - Every diff uses `--no-renames`, so a file moved from `crates/` to `app/` shows its source path.
- [x] `scripts/tests/test_gate_scope.py` covers every mode in throwaway repos and runs `ci` against a fake `gh`. Its guard fails when a gate input names a listed path:
  - every `.rs` file under `crates/` (sources too: `include_str!`), the Python and shell files under `bench/`, `scripts/tests/` and `packaging/`, the scripts those tests run, and the recipes reachable from `gate:` in the Makefile.
  - It looks only at literals anchored at the repo root, after stripping `../` and `./`.
  - Literals that name a temp-tree path carry an allow-list entry with a reason.
  - Every list entry must name a tracked path, so a renamed folder cannot leave a stale exclusion.
- [x] `.githooks/pre-commit` skips `gate-fast` on `skip` or `swift`, with one line saying why. This changes the hook's policy: it now decides on the staged paths, not on the whole working tree.
- [x] `.githooks/pre-push`:
  - `skip`: no gate.
  - `swift`: `make ddckit-test`, which is `swift test` of DDCKit only. The full `make app-test` signs a fixture with `codesign`, which agent sessions must not run.
  - `rust`, or anything else: `make gate`.
- [x] `gate.yml`:
  - **`scope` job:** runs on ubuntu with `fetch-depth: 0` and `actions: read`.
    - It is never `continue-on-error`.
    - Base = the newest commit that is an ancestor of `HEAD` and whose push Gate run concluded `success` with `gate`, `coverage` and `release-build` all `success`. Candidates come from all branches, and ancestry decides. It reads one page of 50 runs and checks at most 20 ancestors.
    - It answers `rust` when there is no base, when any call fails, and for `pull_request` and `workflow_dispatch` runs.
  - **Rust jobs:** `gate`, `coverage`, `release-build` and `check` have `needs: scope` and `if: ${{ !cancelled() && needs.scope.outputs.scope != 'skip' }}`, so a failed `scope` runs them.
  - **`app`** always runs.
  - **`load`** moves to `nightly.yml` as `load-m`.
- [x] `scripts/ci-green.sh --full` is used by `make tag` (`tag-release.sh`), `make dist` and `release.yml`.
  - It also requires a green run of `HEAD` whose `gate`, `coverage` and `release-build` jobs are `success`. That can be a push run or a `workflow_dispatch` run, any of the last 20, not only the newest.
  - When none qualifies, its message gives the way out: `gh workflow run gate.yml --ref main`, wait, rerun. A dispatched run always runs the Rust jobs.
- [x] `Makefile`:
  - `dist: require-tag ci-green-full`, then the build.
  - `fuzz-lock` on stable goes into `gate` after `deny machete`, and `fuzz` depends on it.
  - `ddckit-test`, which `app-test` depends on.
  - `coverage-ratchet` picks the newest green `main` run whose `coverage` job succeeded.
  - The comments that say `dist` runs the gate are corrected, and so is `scripts/require-tag.sh`.
- [x] `fuzz/Cargo.lock` refreshed.
- [x] AGENTS.md "The gate" section (outside the projectstore block) and the README's development section:
  - which pushes skip the Rust tiers;
  - that a green hook on an app-only push says nothing about Rust.
- [x] Vault runbook "Cutting a release: tag, CI, archives, dist and publish":
  - the version bump refreshes `fuzz/Cargo.lock` and carries the release notes in the same commit;
  - `make dist` no longer re-runs the gate.
- [x] Measured after the change: one app-only push and one core push, with the duration of each job, recorded in this story.

## Implementation Plan

Planned with `projectstore:planner` on 2026-10-30. Critiqued by `projectstore:critic` twice on the same day.
  - First pass: the base must require `coverage` and `release-build` too, not only `gate`; releases must not depend on the scope logic; renames; the tag push; the guard's anchoring; the shallow clone; the fuzz lock on stable.
  - Second pass: a way out for `--full` through a dispatched run; deletions skip; a tag beside a branch is judged; `release.yml` uses `--full`.

1. Refresh `fuzz/Cargo.lock`. Add `fuzz-lock` to `gate`.
2. Add `.config/rust-free-paths`, `scripts/gate-scope.sh` and `scripts/tests/test_gate_scope.py`, including the guard and the hook cases.
3. Update the hooks. Add `ddckit-test`.
4. `gate.yml`: add `scope`, gate the Rust jobs on it, and move `load` to `nightly.yml`.
5. `ci-green.sh --full`, `dist`, `tag-release.sh`, `coverage-ratchet` and the comments.
6. AGENTS.md, README and the vault runbook.
7. Push a core change, then an app-only change, and record the job durations.

## Acceptance Criteria

- [x] A push of only Rust-free paths, such as an app Swift file outside `app/Packages/`, behaves as follows:
  - the pre-push hook runs no gate;
  - CI skips `gate`, `coverage`, `release-build` and `check`, and runs `scope` and `app`;
  - the run is green, and its jobs together take under 5 minutes, not counting time queued for a runner.
  - Evidence:
    - Branch `gate-scope-proof` held commit 3f62f09, a comment in `app/DeepDriveCleaner/AppLog.swift`. The branch was deleted after the run.
    - Pre-commit printed "only Rust-free paths staged … gate-fast skipped". Pre-push printed "… gate skipped", and `git push` took 3.5 s.
    - Gate run 41048680391 concluded success. Its `scope` job logged `gate-scope: base 48f9fba (run 41047623783)` and took 0.1 min. `app` took 2.0 min. `gate`, `coverage`, `release-build` and `check` were skipped.
    - The jobs took 2.1 min together, and the run took 2 min of wall time.
- [x] A push touching a path off the list runs the full gate locally and every Rust job in CI. Example paths: a `crates/` file, `docs/scan.schema.json`, a DDCKit fixture, or a file renamed from `crates/` into `app/`.
  - Evidence:
    - The pushes of 3939793 and 48f9fba ran `make gate` in the pre-push hook. Gate runs 41044862295 and 41047623783 ran every Rust job.
    - `Paths.test_paths_a_gate_step_reads_run_the_rust_tiers` covers `docs/scan.schema.json` and the DDCKit fixture.
    - `Push.test_a_crate_change_runs_rust` covers a `crates/` file, and `Push.test_a_rename_out_of_crates_runs_rust` covers the rename.
- [x] `ci` mode, tested with a fake `gh` in `scripts/tests/test_gate_scope.py`:
  - **Fail toward `rust`:**
    - a Rust-free X on Y whose run failed only in `coverage` → `rust`;
    - a `gh` failure, no runs, or no green ancestor → `rust`;
    - a pull request or a dispatch → `rust`.
  - **Skip:**
    - a Rust-free X on a Rust-free W on a green B → `skip`, with B as the base;
    - a candidate whose run is green but whose `gate` was skipped → not a base.
  - Evidence: these `Ci` tests pass in `python3 scripts/tests/test_gate_scope.py`, 39 tests OK:
    - `test_a_base_whose_coverage_failed_is_not_a_base`
    - `test_a_gh_failure_runs_rust`
    - `test_no_runs_run_rust`
    - `test_a_run_on_another_line_is_not_an_ancestor`
    - `test_pull_requests_and_dispatches_run_rust`
    - `test_a_chain_of_app_changes_finds_the_base_two_back` (it asserts `base <B>` on stderr)
    - `test_a_green_run_that_skipped_its_rust_jobs_is_not_a_base`
- [x] `push` mode, tested in throwaway repos:
  - deletion only → `skip`;
  - a tag → `skip`;
  - a tag beside a branch with a `crates/` change → `rust`;
  - a new branch with no new commit → `rust`;
  - a new branch → diffed from the merge base;
  - an unknown remote sha → `rust`;
  - empty stdin → `rust`;
  - a rename from `crates/` to `app/` → `rust`;
  - a DDCKit source change → `swift`.
  - Evidence: these `Push` tests pass, one per case:
    - `test_a_deletion_alone_skips`
    - `test_a_tag_skips`
    - `test_a_tag_beside_a_branch_with_a_crate_change_runs_rust`
    - `test_a_new_branch_with_nothing_new_runs_rust`
    - `test_a_new_branch_is_judged_from_its_merge_base_with_origin_main`
    - `test_an_unknown_remote_sha_runs_rust`
    - `test_empty_stdin_runs_rust`
    - `test_a_rename_out_of_crates_runs_rust`
    - `test_a_ddckit_change_answers_swift`
- [x] The guard fails when a Rust or Python test reads a listed path. This is shown by a temporary edit, which is then reverted.
  - Evidence: on 2026-10-30, the literal `"website/index.html"` was appended to `crates/ddc-core/tests/disk.rs`.
    - `Guard.test_no_gate_input_names_a_rust_free_path` then failed with `[('crates/ddc-core/tests/disk.rs', 'website/index.html')] != []`.
    - The edit was reverted with `git checkout`, leaving the tree clean.
- [x] `make tag` and `make dist` (`ci-green --full`) refuse a `HEAD` whose run skipped the Rust jobs, naming `gh workflow run`, and accept it once a dispatched run of `HEAD` passed them. This is shown by the script's tests with a fake `gh`.
  - Evidence: these `CiGreenFull` tests pass:
    - `test_full_refuses_a_run_that_skipped_its_rust_jobs`, which asserts the `gh workflow run` advice;
    - `test_full_accepts_a_dispatched_run_of_head`;
    - `test_full_passes_a_run_that_ran_them`;
    - `test_without_full_the_run_conclusion_decides`.
- [x] Gate on a core commit is green with no `load` job. Nightly's `load-m` job is green.
  - Evidence:
    - Gate run 41047623783 on 48f9fba concluded success and has no `load` job.
    - Nightly run 41049050995, dispatched on 48f9fba: `load-m` succeeded in 17.7 min, and every other nightly job succeeded too.
- [x] `make -n dist` shows `require-tag`, `ci-green --full` and the build, with no `gate` and no `app-test`.
  - Evidence: `make -n dist` prints `./scripts/require-tag.sh`, `./scripts/ci-green.sh --full`, `./scripts/build-dist.sh` and `./scripts/build-app.sh --dist`, and nothing else.
- [x] `make fuzz-lock` fails on the 0.7.0 lock and passes after the refresh. The next Nightly's `fuzz` job gets past `cargo metadata`.
  - Evidence:
    - With `git show 3939793^:fuzz/Cargo.lock` restored (`ddc-core` 0.7.0), `make fuzz-lock` exits with `make: *** [fuzz-lock] Error 1`. With the committed lock it passes.
    - Nightly run 41049050995: `fuzz` succeeded in 54.3 min. Scheduled run 41034359935 had failed at `cargo metadata`.
- [x] Owner checks: none. Every criterion above is shown by a command or a CI run.

## Final Summary

**What changed.** Every check is kept, but each now runs where it pays.
- M-size `load` moved from `gate.yml` to `nightly.yml` as `load-m`.
- A release runs the gate once, in CI on the release commit. `make dist` and `make tag` require that run through `ci-green --full`.
- A change whose every path is on `.config/rust-free-paths` (the app's own sources, `website/`, `docs/releases/`) skips the Rust tiers in both hooks and in CI. `scripts/gate-scope.sh` is the one classifier, and any doubt runs Rust.
- `fuzz/Cargo.lock` is refreshed to 0.8.0, and `make gate` checks it.

**Measured.**

| Run | Jobs | Wall time |
|---|---|---|
| Before (average of 5 `main` runs) | `load` 16.5, `gate` 6.8, `coverage` 3.6, `app` 2.0, `release-build` 1.5 | about 17 min |
| Core commit 3939793 (run 41044862295) | `scope` 0.1, `gate` 3.3, `coverage` 3.6, `app` 2.4, `release-build` 1.4 | 3.8 min |
| Core commit 48f9fba (run 41047623783) | `scope` 0.1, `gate` 8.3, `coverage` 3.4, `app` 3.2, `release-build` 1.9 | about 8.5 min |
| App-only commit 3f62f09 (run 41048680391) | `scope` 0.1, `app` 2.0, Rust jobs skipped | 2 min |

`gate`'s time varies by run on the hosted macOS runner, from 3.3 to 8.3 min for the same work. Locally, an app-only push takes seconds instead of about 2 minutes.

**Found on the way.** CI's first run on 3939793 failed in `matcher_reference`'s proptest.
- The generator had made `.gradle/Caches` and `.gradle/caches`, which macOS folds into one folder. The bug predates this story; a random seed found it.
- Fixed in 48f9fba: the generator now spells a name the way the tree already does. `names_that_differ_only_in_case_are_one_folder` pins the case.

**Tests executed.**
- `python3 scripts/tests/test_gate_scope.py`: 39 tests OK.
- `make gate` in the pre-push hook of 3939793 and 48f9fba.
- Gate runs 41044862295, 41047623783 and 41048680391.
- Nightly run 41049050995.

**Risks and follow-ups.**
- The M-size correctness check (trees against their manifests, the CLI's exit and the items a clean leaves) now runs on `main` once a night. A fault that shows only at about 10^5 entries is found within 24 hours, not before the push. The owner chose this trade.
- No CI job compiles the app host (`app/DeepDriveCleaner/`), and a GUI push now runs only `app` (DDCKit's `swift test` and the signing check). Follow-up: compile the app target in CI.
- Nightly has no `release-build` (`scripts/build-cli.sh`).
- Under load from parallel agent test runs, about 12 on 2026-10-30, `ddc-core::recheck` hit its 120 s timeout in the pre-push gate twice. The tests pass alone. The pushes went through on a retry.

## Technical Notes

- **Why the base needs three jobs.** Run 41016775256 (`6ae225e`) had `gate` succeed while `coverage` failed. A rule that looked at `gate` alone would have taken it as a base.
- **Why releases do not trust the chain.** `--full` confines any scope bug to development pushes. The version bump touches `Cargo.toml`, so the release commit takes the full path anyway.
- **What the hook trusts.** The pre-push hook takes the remote sha as its base. After a red Rust push, an app-only push on top runs no gate locally. CI's base rule is what catches it.
- **Skipped jobs.** Skipped jobs conclude `skipped`, and GitHub then concludes the run `success`. That is why `ci-green` without `--full` keeps its meaning for development: "green, up to the scope rule". The `check` jobs are continue-on-error, and they fail on purpose until story "Move OS calls behind the platform module" (DDC-007) lands.
- **The `scope` job's cost.** It adds about 30–60 s before the Rust jobs. `app` does not wait for it.
- **`ddckit-test`.** The real-CLI tests in `ddckit-test` use `target/debug/deepdrivecleaner` when it exists and skip themselves when it does not. A stale local binary is possible, so CI's `app` job is the authoritative check.
- **Literals the scripts must never contain.** The string named in `crates/ddc-cli/tests/structure.rs` must never appear in `scripts/`, `.github/workflows/` or `.githooks/`.
- **Where a wrongly green push is caught.** Nightly catches it only on `main`. A feature branch's is caught when it merges into `main` (its range is judged again from the base), or at `make tag`.
- **What no job compiles.** No CI job compiles the app host (`app/DeepDriveCleaner/`). `make app-test` is DDCKit's `swift test` plus a signing check. This was true before this story, but it now matters more, since a GUI push runs only `app`. A follow-up is to compile the app target in CI.
- **Nightly and `release-build`.** Nightly's `make gate` includes `release-check`, but not `scripts/build-cli.sh`.
- **Other runs per push.** The Install workflow still runs on every push. `ci-green` does not wait for it.

## Dependencies

- Story "Mandatory gate before every commit" (DDC-005): the tiers this story schedules.
- Story "Memory and CPU under load: peak memory, leaks and CPU time" (DDC-005): its `load` job moves to Nightly.
- Story "Coverage floor and property tests for the safety invariants" (DDC-005): the `fuzz` job and the coverage ratchet.
- Story "Move OS calls behind the platform module" (DDC-007), in progress in another session: it will touch the Makefile's `gate` and the `check` jobs after this story lands.

## Attachments

-

---

*Last updated: 2026-10-30*

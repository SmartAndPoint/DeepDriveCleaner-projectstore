---
type: story
id: "story-build-matrix-every-target-built-tested-and-packaged-from-one-tag"
epic: "DDC-007"
title: "Build matrix: every target built, tested and packaged from one tag"
status: in-progress
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-31
external_refs: {}
tags: [ci, release, build, linux, windows]
code_refs: [".github/workflows/gate.yml", ".github/workflows/release.yml", "Makefile", "README.md", "crates/ddc-cli/tests/structure.rs", "scripts/build-cli.sh", "scripts/build-dist.sh", "scripts/ci-green.sh", "scripts/publish-release.sh", "scripts/require-tag.sh", "scripts/tag-release.sh", "scripts/check-build-number.sh", "scripts/tests/", "scripts/native-skips.sh", "scripts/gate-scope.sh", ".config/native-skips/", ".config/nextest.toml", ".gitattributes", "crates/ddc-core/src/platform/", "crates/ddc-core/tests/common/mod.rs", "crates/ddc-cli/tests/common/", "crates/ddc-cli/tests/disclaimer_cli.rs", "crates/ddc-loadgen/src/lib.rs"]
specs: []
started_at: "2026-10-30T07:10:03.214Z"
closed_at: null
plan_updated_at: "2026-10-30T16:27:36.938Z"
---

# Build matrix: every target built, tested and packaged from one tag

| Field | Value |
|---|---|
| **Epic** | [DDC-007](../epic.md) |
| **Status** | in-progress |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

This is the epic's backbone, the build. Today one script builds one target (`aarch64-apple-darwin`) on the owner's Mac. The story grows in three stages, so the build exists from the start and does not wait for the port:

1. **Thin slice (after DDC-005's `gate.yml` exists).** If this epic starts first, this story creates a minimal `gate.yml` that DDC-005 then extends.
   - DDC-005's gate workflow on macOS.
   - Jobs that run `cargo check --workspace --target x86_64-unknown-linux-gnu` and `x86_64-pc-windows-msvc`, marked `continue-on-error`.
   - One `cargo build --release` on macOS, so release-only breakage (LTO, codegen units) is caught before `make dist`.
2. **Native test matrix (after story "Move OS calls behind the platform module").** The check jobs become required. The gate runs natively on `ubuntu-24.04` and `windows-latest`: fmt once, clippy and tests per OS. On an unported platform only the platform-neutral tests run, because the CLI refuses scans there (seam ADR).
3. **Release archives (per platform, after its catalog story).** On a release tag, `release.yml` builds the Linux and Windows archives on native runners. It requires the `gate.yml` run for the same commit to be green, so every packaged binary passed the test suite on its own OS. It smoke-tests each built binary, attests its provenance, and uploads the archives as workflow artifacts. `publish-release.sh` collects them with the owner's locally signed macOS archive and the app zip into one release.

Targets:

| Target | Built on | Archive | Trust |
|---|---|---|---|
| `aarch64-apple-darwin` | owner's Mac (as today) | `deepdrivecleaner-<v>-aarch64-apple-darwin.tar.gz` | Developer ID signature, notarized |
| `x86_64-unknown-linux-musl` | `ubuntu-24.04` | `deepdrivecleaner-<v>-x86_64-unknown-linux-musl.tar.gz` (static) | provenance attestation |
| `aarch64-unknown-linux-musl` | `ubuntu-24.04-arm` | `deepdrivecleaner-<v>-aarch64-unknown-linux-musl.tar.gz` (static) | provenance attestation |
| `x86_64-pc-windows-msvc` | `windows-latest` | `deepdrivecleaner-<v>-x86_64-pc-windows-msvc.zip` | provenance attestation; signing per story "Windows install: PowerShell script, winget and a signed binary" |

`SHA256SUMS` covers every archive in the release. A locally built macOS archive cannot carry a CI attestation; its signature and notarization are its proof.

### Release flow

Today `publish-release.sh` creates the tag itself (`gh release create --target <built commit>`). It refuses a tag that already exists on origin without a release (`STRAY_TAG`), and it treats an existing release as "resume". A workflow that creates the release itself would collide with both.

The flow becomes:
1. The owner (or `make tag`) pushes an annotated `v<version>` tag at `HEAD` of `origin/main`.
2. `release.yml` runs on the tag. It builds, smoke-tests and attests, and uploads the archives as **workflow artifacts**. It never creates the GitHub release.
3. `make dist` builds and notarizes the macOS archive and the app zip from the same commit. `build-dist.sh` checks that `git ls-remote origin refs/tags/v$VERSION` points at `HEAD`. It asks origin, as `publish-release.sh` already does, because local tags are not fetched in this clone.
4. `publish-release.sh` checks:
   - the tag exists on origin and points at `HEAD`;
   - the `release.yml` run for that commit succeeded, when the `enabled` list is non-empty; otherwise only the tag check applies.
   
   It fetches the archives with `gh run download`, writes `SHA256SUMS` over everything, and creates the release as today. The `STRAY_TAG` refusal becomes the checks above.
5. If `release.yml` fails, the tag exists without a release, which is the state `STRAY_TAG` used to prevent. Recovery: fix on `main`, delete the tag on origin (it has no release yet), and tag again, or bump the version. While targets are enabled, `publish-release.sh` refuses while the run is red.

## Decomposition

- [x] Stage 1: `gate.yml` gains the `cargo check` jobs for Linux and Windows (`continue-on-error`) and a macOS `cargo build --release` job. — evidence: commit 0a88802 "ci: build matrix stage 1 and a tag-first release flow"
- [x] Stage 2: make the Linux and Windows jobs required, and run the gate natively on both. Test harness portability is done in story "Move OS calls behind the platform module".
  - Core tests of the gated entry points (`report::scan*`, `scan::walk`, `disk::measure*`, `exec::execute`, `preflight`, `free::run`) skip through `crates/ddc-core/tests/common/mod.rs` `unsupported()` on a native job of a platform not yet supported (ADR "Platform seam: one platform module per OS with declared capabilities", §9). Story "Platform seam: ADR and capability model" added the helper; story "Move OS calls behind the platform module" applies it, in its test-harness item, to the core tests and to the CLI integration tests. Stage 2 waits for that.
  - Evidence: PR #19 (head a8fc79d, rebase-merged as 0b6d327); `main`'s Gate run 41125669660 green on all seven jobs; ruleset 26206449 requires `check-x86_64-unknown-linux-gnu` and `check-x86_64-pc-windows-msvc`. See "Stage 2 outcome" below.
- [x] `release.yml` on `v*` tags, created in stage 1 with an empty `enabled` list, so the catalog stories only add to it. A `plan` job emits the matrix and the build job carries an `if:` on a non-empty list, because an empty matrix fails the run while a skipped job does not. One job per enabled target:
  - waits for, or requires, a green `gate.yml` run for the same commit;
  - `cargo build --release --locked --target <t>` with `--remap-path-prefix` as in `build-dist.sh`;
  - smoke test: `--version` equals the tag, and `scan --json` on a small generated tree validates against the schema;
  - package with `LICENSE.md`, `DISCLAIMER.md` and `README.md`, flat as the macOS tarball;
  - check that no runner path or home folder appears in the binary;
  - `actions/attest-build-provenance`;
  - upload as a workflow artifact.
  - Evidence: commit 0a88802: `.github/workflows/release.yml` (`RELEASE_TARGETS: '[]'`, `scripts/ci-green.sh --full`, the `--version` and `scan --json` smoke test, the flat archive, "No runner paths in the archive", `actions/attest-build-provenance@v4`, `actions/upload-artifact@v7`). It ran on the v0.9.0 and v0.10.0 tags (runs 41075547893, 41115665055): `plan` ran and `build` was skipped on the empty list. It has not run with a target enabled yet. That is stage 3's AC, and it waits for the first `RELEASE_TARGETS` entry from story "Linux scanner and rule catalog", so this story stays in progress until then.
- [ ] A per-target `enabled` list in `release.yml`: Linux and Windows are added by their catalog stories, in the same commit that flips the platform's `SUPPORTED` switch. (The list, `RELEASE_TARGETS`, exists since 0a88802 and is empty; the entries land with the catalog stories.)
- [x] `make tag`: refuses unless the tree is clean, `HEAD` equals `origin/main`, and the version in `Cargo.toml` matches. Then creates and pushes the annotated tag. — evidence: commit 0a88802, `scripts/tag-release.sh` and the Makefile's `tag:` target
- [x] `build-dist.sh`: require the tag on origin at `HEAD` (`git ls-remote`). — evidence: commit 0a88802, `scripts/require-tag.sh`, called from `scripts/build-dist.sh`
- [x] `publish-release.sh`: the checks and download above, `SHA256SUMS` over all assets, and refusal when an enabled target's artifact is missing. — evidence: commit 0a88802, `scripts/publish-release.sh` (`RELEASE_TARGETS`, `SHA256SUMS`); its run on a real pre-release is stage 3's AC
- [x] The release runbook (or the Makefile header): tag, wait for `release.yml`, `make dist`, publish, and the recovery from a failed `release.yml`. Who pushes the tag is stated. — evidence: runbook "Cutting a release: tag, CI archives, dist and publish" (`ops/`), "Who pushes the tag: the owner, with `make tag`"

## Implementation Plan

Stage 1 lands now; stages 2 and 3 follow the stories they depend on.

1. `scripts/build-cli.sh <target>`: the release build with `--remap-path-prefix`, then the unsigned checks (architecture, `--version`, no home, checkout or `CARGO_HOME` path, the `nm` and `DDC_OP_STATS` checks). `build-dist.sh`, the gate's release build and `release.yml` all call it.
2. `gate.yml`: push trigger on branches only, so a tag push does not start a second Gate run. A `check` matrix (`x86_64-unknown-linux-gnu` on `ubuntu-24.04`, `x86_64-pc-windows-msvc` on `windows-latest`, native `cargo check --workspace --locked`, `continue-on-error`). A `release-build` job on `macos-26` running `build-cli.sh aarch64-apple-darwin`.
3. `release.yml` on `v*` tags with `RELEASE_TARGETS: '[]'`. A `plan` job checks the tag against the Cargo version and emits the matrix and `any`; the `build` job runs only when `any` is true: wait for the Gate run, build, smoke test (`--version`, `scan --json` against the schema), flat archive, runner-path check, provenance attestation, upload as a workflow artifact.
4. `ci-green.sh` takes an optional workflow file (default `gate.yml`).
5. `scripts/tag-release.sh` and `make tag`: clean tree, on `main`, `HEAD` equals `origin/main`, Cargo version equals the app's `MARKETING_VERSION`, release notes exist, tag absent locally and on origin; then an annotated tag, pushed.
6. `build-dist.sh` requires the peeled tag on origin at `HEAD`.
7. `publish-release.sh`: "tag on origin at `HEAD`" replaces `STRAY_TAG`; reads `RELEASE_TARGETS` fail-closed; with targets enabled, requires a green `release.yml` run, downloads each archive (missing means refuse), checks its listing and attestation. `SHA256SUMS` (`<hex>  <name>`, sorted by name with `LC_ALL=C`, no entry for itself) over every asset; `gh release create --verify-tag`; a digest check per asset. A version with `-` is a pre-release (no cask, no tap).
8. The Makefile and README give the order: `make tag` → `release.yml` → `make dist` → publish. The vault runbook "Cutting a release: tag, CI archives, dist and publish" gives the steps and the recovery.

### Stage 2 plan (2026-10-30)

Stage 2 starts now. Story "Move OS calls behind the platform module" is done, and since PR #4 the native jobs run `cargo test --workspace --no-fail-fast`.

**Baseline.** Gate run 41082747167 on 642cf39 failed 36 tests on Ubuntu and 59 on Windows, and none in the macOS gate. PR #10 fixed one Windows failure after that run. The implementation re-baselines on the first run of its own branch and works from that list.

**The rule.** Every test either passes natively or skips with a printed reason. There is no third kind.

**Skip** when a test exercises something not ported. There are four channels, all decided at run time from `PLATFORM`, so none skips on macOS:
- `unsupported()` (`ddc-core/tests/common`, `ddc_loadgen::unsupported`, the CLI tests' `common`): an entry point that `require_supported` refuses, or platform code that is unported today and portable later. Examples: the bulk reader (`bulk_reader_matches_lstat_per_entry_across_batches`), `fsops`-backed readers such as `report::backups`, `exec::delete`, `exec::system`.
- `lacks(Capability::X)`, a new form beside it: a test of a capability the platform does not declare. Examples: Homebrew (`brew.rs`), WholeDisk (`disk_free.rs`), CodingAgents (`agents.rs` session tests). It stays meaningful after a flip: a supported Linux without Homebrew still skips the brew tests. Capabilities are only those of the closed `Capability` enum; nothing new is added.
- In-crate unit tests (`#[cfg(test)]` modules under `src/`) cannot reach `tests/common`. A `#[cfg(test)] pub(crate) fn skipped(test: &str) -> bool` beside `require_supported` in `ddc-core/src/platform/mod.rs` prints the same line; loadgen's in-crate tests call the existing `crate::unsupported`. This covers the unit-test failures in `agents/running.rs`, `exec/system.rs`, `exec/delete.rs`, `report/backups.rs`, `free/probe.rs`, loadgen `leaks.rs` and `measure.rs`. `agents/claude.rs` is the overflow fix below, not a skip.
- Tool dependencies of tests.
  - In-source unit tests use `platform::testing`, which is `#[cfg(test)]` and already holds `symlink` (`macos/testing.rs`, `unported.rs`). It returns `Unsupported` where the tool is missing, and the test skips on that. Example: `/usr/bin/plutil` for the binary-plist backups tests, which would otherwise hard-fail after the Linux flip.
  - Integration and CLI tests cannot reach `platform::testing`: it is test-only, and `ddc-cli` has no library. They keep their `tests/common` twins (`common::symlink`, a `script` driver). Those twins may use an OS `cfg`, as ADR §9 allows and as `structure.rs`, which scans only `src/`, permits. A twin returns `io::Result<()>`, and the test skips on `ErrorKind::Unsupported`. A twin that still panics (today's `common::symlink`) is only called after the test has skipped through `unsupported()` or `lacks()`.

**Fix** when the failure is in portable code or in the harness.
- **Line endings.**
  - Windows checks out text files with CRLF. That turns every `committed_*_schema_is_current` stale. It also fails `structure.rs` on `Cargo.toml`'s `[lints]` block and on the Makefile's `target/unported` line (`only_the_macos_module_and_the_counters_allow_unsafe_code`, `only_the_unported_check_builds_the_simulated_platform`).
  - A global `.gitattributes` fixes this: `* text=auto eol=lf`, with the fuzz regression inputs `-text` and images `binary`.
  - `git ls-files --eol` shows no CRLF file in the index, so no renormalization commit is needed. That includes `*.pbxproj`, `.strings` and `.jsonl` fixtures.
- **Absolute paths.**
  - On Windows, `/Users/...` is not absolute (`Path::is_absolute` needs a drive). The parsers keep native semantics: `ListedItems` (`plan.rs` `normalised`), settings (`settings.rs`), and the `agents::running` and `free::probe` filters.
  - So macOS-shaped data does not parse on Windows. Until story "Windows scanner and rule catalog" adds Windows-shaped inputs, the tests that feed it skip through `unsupported()`: `a_malformed_list_is_refused_before_anything_runs`, `running.rs` and `probe.rs`.
  - The two fuzz-regression tests keep their no-panic half on every OS: every kept input still runs. Only the "at least one input is accepted" assertion is gated, because Windows path handling is where a new panic would appear.
  - That story records the debt. So does `exec::system`'s test, which spawns `/bin/sleep` literally.
- **The overflow.** The `agents/claude.rs` test computes `procs::uid() + 1`, which overflows when the stub's uid is `u32::MAX`. It uses `wrapping_add`.
- **The load generator.**
  - `real_home()` is `None` off macOS, and the generator builds macOS trees by design (catalog split). Its tests skip through `ddc_loadgen::unsupported`, `isolation.rs` included.
  - `tests/guard.rs` keeps its `#![cfg(unix)]`, which ADR §9 allows for tests that are OS-specific by nature. Its body gains the same runtime skip, so it stops failing on Ubuntu.
- **The disclaimer tests** check product behaviour on every platform, so they are not skipped on Linux.
  - The harness drives util-linux `script` with its own argument order on Linux (`script -qec "<cmd>" /dev/null`) and keeps BSD's on macOS. The test file's driver picks with `cfg!(target_os = ...)`.
  - Windows skips. A ConPTY harness is debt for the Windows story.
- **Missing guards.** `there_is_no_yes` (`disk_free_cli.rs`) lacks the `unsupported()` guard its sibling has. The PR greps for every `Fx::new()` caller, and every other entry-point fixture, that has no guard.

**Make skips observable.** `cargo test` hides a passing test's output, so a skip line never reaches the log today. Instead:
- The native jobs run nextest pinned to `tools.mk`'s version (`tool: nextest@0.9.150`), with the gate's `ci` profile. That adds the profile's slow-test timeout and termination, and `success-output = "immediate"` goes into `[profile.ci]`.
- nextest does not run doctests, so each leg also runs `cargo test --workspace --doc --locked --target <t>`.
- A step after the tests compares the `skipped <test>:` lines with a committed list, `.config/native-skips/<os>.txt`. It is keyed by OS, not by triple, so stage 3's extra Linux targets share one list.
  - It runs with `shell: bash`; the Windows default is pwsh.
  - The comparison is `LC_ALL=C sort -u`, then `diff -u`, so the log shows the change.
  - A test checks that every skip label equals its enclosing fn name, so a stale or duplicate label cannot hide a new skip.
  - A new skip, or a listed one that no longer skips, fails the job with the difference.
  - The counts go to `$GITHUB_STEP_SUMMARY`.
- A new skip is therefore a visible diff in a PR.
- On macOS, `ddc-core/tests/platform.rs` `macos_is_supported_with_every_capability` (`SUPPORTED`, `CAPABILITIES == Capability::ALL`) is the guarantee that nothing skips, since every channel above decides from `PLATFORM`.

**CI and the ruleset.**
- The matrix legs get stable names: `name: check-${{ matrix.target }}`, giving `check-x86_64-unknown-linux-gnu` and `check-x86_64-pc-windows-msvc`. A later runner-label bump then does not rename a required check.
- Windows pins `windows-2025`, as Ubuntu pins `ubuntu-24.04`.
- `continue-on-error` goes. The gate.yml header and job comments, the README and Makefile gate sections, and the vault runbook "Cutting a release: tag, CI archives, dist and publish" stop calling the legs advisory.
- `scripts/ci-green.sh` adds the two names to its required list (5 jobs, not 3). `scripts/gate-scope.sh` adds them to `BASE_JOBS`, and its fixtures in `scripts/tests/test_gate_scope.py` follow.
- The two contexts are added to the ruleset by API, because the UI lists only names it has recently seen.
- A skipped leg, on a Rust-free push, satisfies a required check as `gate` does today.
- A flaky required leg is re-run with `gh run rerun --failed`. Profile `ci` has no retries, so a flaky test is fixed, not retried.

**Order.**
1. Harness:
   - `.gitattributes`;
   - the skip channels (`lacks`, the in-crate `skipped`, the `platform::testing` tool functions);
   - the nextest native step, the skip-list check and the stable job names.

   One commit each.
2. Iterate on a pushed branch, since the native legs run only in CI:
   - gate or fix by area: core unit, core integration, CLI, loadgen;
   - commit the skip lists from the run where everything else is green.
3. Remove `continue-on-error`, then update ci-green.sh, gate-scope.sh and the docs.
4. Before opening the PR, add the two required checks to ruleset 26206449.
   - Auto-merge then waits for both native legs on the PR's own run.
   - An in-flight PR from another session is blocked until it rebases onto the fix. Tell that session.
   - The ruleset change is reversible, so if the PR stalls, the checks come off again.
5. Confirm on the next push to `main` that all three are green and required.

ADR "Platform seam: one platform module per OS with declared capabilities" §9 gains an amendment: the capability form `lacks()`, the in-crate `skipped()`, tool helpers (`platform::testing` in-source, `tests/common` twins elsewhere), and the committed skip lists.

### Stage 2 outcome (2026-10-31)

PR #19 merged. The plan held, with these differences:
- **A skipped matrix job posts no check names.** When a job-level `if:` skips a matrix job, GitHub does not expand it, so a Rust-free push would never report `check-<target>`, and a PR requiring them would wait for good. The `check` job now always runs and the scope decides each step instead. A Rust-free run (41121811937) posts both names as success. It still skips gate, coverage and release-build, so it is never a scope base, and `ci-green --full` refuses it. The plan's line "a skipped leg satisfies a required check as `gate` does today" holds only for non-matrix jobs.
- **Windows temp folder.** With `shell: bash`, the runner's temp folder lies inside `HOME`, and the test trees' guard refused 11 integration tests. The Tests step sets `TMP` and `TEMP` to `runner.temp`.
- **Hand-printed skip lines.** `no_system_process_owns_a_cache` printed two unlabelled `skipped:` lines that neither the list compare nor the label test saw. It now fails when it has nothing to check. `every_skip_names_its_own_test` also refuses any `eprintln!("skipped` outside a helper that takes `test`.
- **The overflow fix** was dropped in the rebase: `main` now injects a fixed uid into that test.
- **Main's new tests during the stage**:
  - four tests in `keep`, `agents::running`, `agents::liveness` and `agents::codex` skip through `platform::skipped`; codex skips only its real-reader tail;
  - eleven skips that main's new tests already printed (`ddc keep remove`, finding dates, the liveness facts) and five from `agent_rules.rs` joined both lists.
- **Lists at merge:** `linux.txt` 431 names, `windows.txt` 417. In `main`'s run 41125669660, Linux ran 649 tests and Windows 617, with no failures. The lists grow until each platform's flip: every new test that skips off macOS adds a line, and within hours of the merge both lists had grown by seven. A new test prefers platform-neutral fixtures. When it must skip, its line names an unported reader, a macOS-shaped input or an absent capability, and the reviewer of that PR sees the line. The file, not a count, is the baseline.
- **A flaky native leg** blocks every merge now that the legs are required, and profile `ci` has no retries. Re-run it with `gh run rerun --failed`, then fix the test.
- **Runners:** `ubuntu-24.04` and `windows-2025`.
- **`there_is_no_yes` and the macOS-path tests also skip on Linux, where they pass.** They return with the Linux flip. The reviewer's open question was whether to check path shape instead of `unsupported()` to keep Linux coverage now; it is left to story "Linux scanner and rule catalog".
- **Faster Linux iteration.** A CI round, counting the local pre-push gate, the queue and the run, took about half an hour. The Linux leg was rerun on an Ubuntu 24.04 x86_64 machine over ssh, in `rust:1.99.1-bookworm` as a non-root user, with the same nextest profile and `scripts/native-skips.sh`, in about a minute, and its counts matched CI's (641 tests, 426 skips on the same tree). Its image has glibc 2.36, older than the runner's 2.39. Once the Linux flip un-skips the statfs and /proc tests, CI stays the judge. Story "Linux scanner and rule catalog" iterates there. Windows still runs only in CI.

## Acceptance Criteria

- [x] Stage 1: on `main`, the macOS gate job is green; the Linux and Windows check jobs run (red is allowed until stage 2); the macOS release build job is green. — evidence: Gate run 41082747167 on 642cf39: gate and release-build green, both check legs ran (red, as allowed)
- [x] Stage 2: every push runs the gate on macOS, Ubuntu and Windows, and all three are required and green on `main`. As built, a push whose scope is Rust runs the macOS gate job, while the Ubuntu and Windows legs run clippy, nextest (profile `ci`), doc tests and the skip-list compare. `make gate`'s other tiers (deny, machete, msrv, leaks and the rest) run on macOS only. — evidence: `main`'s Gate run 41125669660 on 0b6d327 (PR #19): scope, gate, coverage, app, release-build, `check-x86_64-unknown-linux-gnu` and `check-x86_64-pc-windows-msvc` all green; ruleset 26206449 lists all seven as required; Rust-free run 41121811937 posted both native checks as success
- [ ] Stage 3: a pre-release test tag produces the enabled targets' archives, each smoke-tested on its runner; `gh attestation verify <archive> --repo SmartAndPoint/DeepDriveCleaner` passes for each CI-built archive.
- [ ] `publish-release.sh` on that tag produces one release with the CI archives, the macOS archive, the app zip and `SHA256SUMS`, and refuses when an enabled target's artifact is missing or the run failed (tested on a throwaway pre-release).
- [ ] No archive contains a runner path or a home folder (checked in the workflow).

## Final Summary

## Technical Notes

- Native runners rather than cross-compiling (`cargo-zigbuild`, `cargo-xwin`) let each binary run its tests on its own OS. Cross-compiling stays a fallback for building without CI.
- `ubuntu-24.04-arm` is generally available for public repositories (GitHub changelog, 2025-08-07). Re-check runner labels and terms when implementing.
- musl's allocator is slow under multi-threaded allocation. If Linux load checks miss by more than 2×, consider `mimalloc` for musl targets (story "Linux scanner and rule catalog").
- Signing stays with the owner. The workflow holds no Apple credentials.

## Dependencies

- DDC-005 story "Mandatory gate before every commit" (the workflow it extends).
- Story "Move OS calls behind the platform module" (stage 2).
- Stories "Linux scanner and rule catalog" and "Windows scanner and rule catalog" (stage 3 per platform).

## Attachments

-

---

*Last updated: 2026-10-30*

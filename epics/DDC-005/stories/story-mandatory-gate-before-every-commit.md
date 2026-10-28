---
type: story
id: "story-mandatory-gate-before-every-commit"
epic: "DDC-005"
title: "Mandatory gate before every commit"
status: done
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [rust, ci, quality, git-hooks]
code_refs: [".config/nextest.toml", ".config/tools.mk", ".githooks/pre-commit", ".githooks/pre-push", ".github/workflows/gate.yml", ".github/workflows/nightly.yml", "Makefile", "deny.toml", "scripts/ci-green.sh", "crates/ddc-core/Cargo.toml"]
specs: []
started_at: 2026-10-28
closed_at: 2026-10-28
plan_updated_at: 2026-10-28
---

# Mandatory gate before every commit

| Field | Value |
|---|---|
| **Epic** | [DDC-005](../epic.md) |
| **Status** | done |
| **Priority** | p1 |
| **Assignee** | Evgenii Konev |

---

## Description

`make check` (fmt, clippy, test) exists, but nothing runs it unless someone remembers to. Agents commit through `/sp` many times a day, and the owner releases without approval gates. A check that can be forgotten will be forgotten.

This story turns the check into a gate:
- one command defines it;
- a versioned git hook runs it locally;
- a GitHub Actions workflow runs it again for anything that reached GitHub.

The gate has tiers, so a commit stays fast and nothing is skipped:

| Tier | When | Budget | Contents |
|---|---|---|---|
| fast | pre-commit | ≤ 60 s warm | `cargo fmt --all --check`; `cargo clippy --workspace --all-targets --features ddc-core/op-stats --locked -- -D warnings` (not `--all-features`: `op-stats` and `dhat-heap` refuse to coexist; the full tier also lints `--features ddc-core/dhat-heap`); `cargo nextest run --workspace --features ddc-core/op-stats` (with a per-test timeout that turns a hang into a failure; the feature compiles the complexity counters and their tests); doc tests |
| full | pre-push, `make dist` | ≤ 5 min warm | the fast tier, plus: `cargo doc --no-deps` with `RUSTDOCFLAGS=-D warnings`; `cargo deny check` (advisories, licenses, bans, sources); `cargo machete` (unused dependencies); the MSRV check; `cargo build --release --locked` |
| CI | every push and pull request | runner time | the full tier, plus coverage and the M-size load checks |
| nightly | scheduled CI | runner time | L-size load, fuzzing, AddressSanitizer on `sys/` |

Later stories add their checks to these tiers: the S-size load checks and the complexity guard go to the fast tier if they fit its budget, or to the full tier. DDC-006 adds its shape checker.

## Decomposition

- [x] Time today's `make check` on the reference machine (cold and warm), to set the budgets.
- [x] `make gate-fast` and `make gate` (full) in the Makefile. `make check` stays as an alias of `gate-fast`, so nothing that calls it breaks.
- [x] `make tools`: installs the pinned tool versions with `cargo install --locked` (cargo-nextest, cargo-deny, cargo-machete, cargo-llvm-cov). The versions sit in one place.
- [x] `deny.toml`: allowed licenses compatible with PolyForm Noncommercial distribution; deny yanked and vulnerable crates; warn on duplicate versions.
- [x] `cargo machete`: an ignore list in `Cargo.toml` metadata for dependencies used only through macros or features, each with a comment. (None needed: `cargo machete` flags nothing today.)
- [x] `.config/nextest.toml`: slow-test warning at 30 s, timeout at 120 s; a CI profile with retries 0. Nextest does not run doc tests, so the gate runs `cargo test --doc` separately.
- [x] MSRV: either `cargo +1.89 check --workspace` in the full tier, or raise `rust-version` to the pinned toolchain. Decide, and record the decision in the epic's Open Questions.
- [x] `.githooks/pre-commit` (fast tier) and `.githooks/pre-push` (full tier). Both set `PATH=$HOME/.cargo/bin:$PATH` first. `make hooks` sets `git config core.hooksPath .githooks`. The README's development section and AGENTS.md (outside the projectstore block) both say to run it once, so agent clones get the hooks too.
- [x] `.github/workflows/gate.yml`: on push and pull request, on `macos-latest`, run `make tools` (pinned versions, cached by a key over the version list), then `make gate`, coverage and the M-size load checks, with a cargo cache. `.github/workflows/nightly.yml`: the nightly tier. DDC-007's build matrix grows from these files.
- [x] `make dist` depends on `gate` instead of `check`, and refuses a commit whose CI run for `HEAD` is not green (`gh run list --commit`).
- [x] Prove the gate catches each failure kind (see Acceptance Criteria) on a throwaway branch.

## Implementation Plan

Planned with `projectstore:planner` on 2026-10-28.

1. `.config/tools.mk` holds the pinned tool versions: cargo-nextest 0.9.150, cargo-deny 0.20.2, cargo-machete 0.9.2, cargo-llvm-cov 0.9.1. The Makefile includes it, and CI keys its tool cache on it.
2. `ddc-core` declares the features `op-stats` and `dhat-heap`, both empty for now, so the gate commands are already in their final form. The complexity story and the memory story fill them.
3. The Makefile gives each step its own target, so make's `*** [target]` line names the step that failed:
   - `fmt-check`, `clippy`, `nextest`, `doctest`, `clippy-dhat`, `doc`, `deny`, `machete`, `msrv` and `release-check`.
   - `gate-fast` runs `require-tools`, `fmt-check`, `clippy`, `nextest` and `doctest`.
   - `gate` runs `require-tools`, then `deny` and `machete`, then `gate-fast`, then `clippy-dhat`, `doc`, `msrv` and `release-check`.
   - `check` is an alias of `gate-fast`.
   - `tools`, `hooks` and `coverage` (lcov under `target/`).
   - `ci-green` runs `scripts/ci-green.sh`.
   - `dist` runs `ci-green`, then `gate`, then `app-test`.
4. `.githooks/pre-commit` runs `gate-fast` and `.githooks/pre-push` runs `gate`. Both put `~/.cargo/bin` first in `PATH`, read nothing from stdin, and do not require a clean tree.
5. `.config/nextest.toml` warns at 30 s and kills a test at 120 s. Its `ci` profile has no retries and does not fail fast.
6. `deny.toml`:
   - licenses MIT, Apache-2.0 and Unicode-3.0;
   - the workspace's own crates ignored;
   - yanked crates denied;
   - duplicate versions warned;
   - crates.io as the only source.
7. Fix the four rustdoc errors that `RUSTDOCFLAGS=-D warnings` raised: two private links, a bare URL and an ambiguous `scan` link.
8. `.github/workflows/gate.yml` runs on `macos-26` and has three jobs:
   - `gate`: `make tools`, then `make gate`;
   - `coverage`: `make coverage`, with the lcov file uploaded;
   - `app`: builds the debug CLI, then runs `make app-test`.
   
   `nightly.yml` runs `make gate` and `make coverage` on `main` at 03:00, and has comment slots for L-size load, fuzzing and AddressSanitizer.
9. The README's development section and AGENTS.md (outside the projectstore block) say to run `make tools` and `make hooks` once per clone, and to never use `--no-verify`.

## Acceptance Criteria

- [x] `make gate-fast` and `make gate` exist. Their timings on the reference machine are recorded in this story: `gate-fast` ≤ 60 s warm and `gate` ≤ 5 min warm, or the epic's open question is re-decided with the measured numbers.
- [x] With the hooks installed, a commit with an unformatted file, a clippy warning or a failing test is refused, with the failing command named in the output.
- [x] A push with a vulnerable or license-incompatible dependency (test with a throwaway branch) is refused by `cargo deny`.
- [x] A test that sleeps forever fails by timeout instead of hanging the gate.
- [x] The GitHub Actions run on `main` is green, and a deliberately broken branch shows a red run. The nightly workflow has run at least once.
- [x] `make dist` runs the full gate first.

## Final Summary

Done on 2026-10-28. The commits on `main`:
- `88b323a`: the gate itself;
- `b0a29c3`: fixes `make tools` on GitHub's `macos-26` image of 2026-10-21, which installs rustup with Homebrew and has no `~/.cargo/bin/cargo`;
- `04c421a`: every pushed commit keeps its own Gate run. Before it, a later push replaced the pending run of an earlier one, and `make dist` reads the run of its HEAD.

Reviewed by `projectstore:reviewer`, and all its findings are fixed:
- the CI tool cache had broken `make tools`; the install now checks the binary's own version;
- `*.profraw` is ignored;
- `ci-green.sh` reads only push runs and splits fields on `|`;
- `.NOTPARALLEL` is set;
- AGENTS.md now asks for a 10-minute tool timeout on commit and push.

GitHub Actions (`SmartAndPoint/DeepDriveCleaner`):

| Run | Commit | Result |
|---|---|---|
| Gate 40719961003 | `88b323a` | green. Cold: gate 12 min, coverage 11 min, app 2 min |
| Nightly 40723880175 (dispatch) | `88b323a` | green |
| Gate 40725327474 | `gate-proof-red` (a failing test) | red at `make: *** [nextest]` and `[coverage]`; app green. Branch deleted |
| Gate 40727061101 | `c164663` | green |
| Gate 40728055440 | `04c421a` | green. Warm cache: gate 6 min, coverage and app 1 min each |

Timings on the reference machine (10 cores):

| Run | Time |
|---|---|
| `make check` before this story | 68 s (partly cold) |
| `make gate` cold, including `make tools` | 170 s |
| `make gate-fast` warm, no change | 42 s |
| `make gate` warm, no change | 50 s |
| `make gate-fast` after editing `ddc-core/src/lib.rs` | 78 s |
| `make gate` after that | 97 s |
| `make gate` after the review fixes | 85 s |
| nextest alone | 39–41 s, 317 tests |

The longest tests are four `ddc-core::recheck` tests: about 35 s each under parallel load, about 5 s each alone. They set the floor of `gate-fast`.

Acceptance evidence (throwaway branch `gate-proof`, hooks installed with `-c core.hooksPath=.githooks`):

| Failure | Result |
|---|---|
| Unformatted file | refused, `make: *** [fmt-check] Error 1` |
| Clippy warning (`x == x`) | refused, `make: *** [clippy] Error 101` |
| Failing test | refused, `make: *** [nextest] Error 100`, `FAIL ddc-core::recheck gate_proof_fails` |
| Test looping forever | `TIMEOUT [120.023s] ddc-core::recheck gate_proof_hangs` |
| `smallvec = "=1.6.0"` | `.githooks/pre-push` refused: `RUSTSEC-2021-0003`, `advisories FAILED` |
| `html2md` (GPL-3.0+) | `.githooks/pre-push` refused: `error[rejected]: failed to satisfy license requirements` |
| `make -n dist` | runs `scripts/ci-green.sh` first, then the full gate, then `swift test`, then `build-dist.sh` |

Known gap: CLI integration tests call `env_clear()`, which drops `LLVM_PROFILE_FILE`. The CLI children's coverage is lost, and they write `*.profraw` into their working folder (now ignored). Story "Coverage floor and property tests for the safety invariants" passes the variable through.

## Technical Notes

- The Makefile forces `~/.cargo/bin/cargo`, because Homebrew's Rust 1.81 sits earlier in `PATH` on the reference machine. Its clippy silently reports nothing. The hooks must do the same.
- Hooks run in agent sessions too. A hook that prompts, or that needs a TCC grant, would block an agent. The gate only reads the repo and temporary folders.
- `.git/info/exclude` hides `codex.json` so `build-dist.sh`'s clean-tree check passes. The hooks must not depend on the tree being clean.

## Dependencies

- None inside the epic. The other stories add to this gate.

## Attachments

-

---

*Last updated: 2026-10-28*

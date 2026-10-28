---
type: story
id: "story-mandatory-gate-before-every-commit"
epic: "DDC-005"
title: "Mandatory gate before every commit"
status: planned
priority: p1
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [rust, ci, quality, git-hooks]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Mandatory gate before every commit

| Field | Value |
|---|---|
| **Epic** | [DDC-005](../epic.md) |
| **Status** | planned |
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

- [ ] Time today's `make check` on the reference machine (cold and warm), to set the budgets.
- [ ] `make gate-fast` and `make gate` (full) in the Makefile. `make check` stays as an alias of `gate-fast`, so nothing that calls it breaks.
- [ ] `make tools`: installs the pinned tool versions with `cargo install --locked` (cargo-nextest, cargo-deny, cargo-machete, cargo-llvm-cov). The versions sit in one place.
- [ ] `deny.toml`: allowed licenses compatible with PolyForm Noncommercial distribution; deny yanked and vulnerable crates; warn on duplicate versions.
- [ ] `cargo machete`: an ignore list in `Cargo.toml` metadata for dependencies used only through macros or features, each with a comment.
- [ ] `.config/nextest.toml`: slow-test warning at 30 s, timeout at 120 s; a CI profile with retries 0. Nextest does not run doc tests, so the gate runs `cargo test --doc` separately.
- [ ] MSRV: either `cargo +1.89 check --workspace` in the full tier, or raise `rust-version` to the pinned toolchain. Decide, and record the decision in the epic's Open Questions.
- [ ] `.githooks/pre-commit` (fast tier) and `.githooks/pre-push` (full tier). Both set `PATH=$HOME/.cargo/bin:$PATH` first. `make hooks` sets `git config core.hooksPath .githooks`. The README's development section and AGENTS.md (outside the projectstore block) both say to run it once, so agent clones get the hooks too.
- [ ] `.github/workflows/gate.yml`: on push and pull request, on `macos-latest`, run `make tools` (pinned versions, cached by a key over the version list), then `make gate`, coverage and the M-size load checks, with a cargo cache. `.github/workflows/nightly.yml`: the nightly tier. DDC-007's build matrix grows from these files.
- [ ] `make dist` depends on `gate` instead of `check`, and refuses a commit whose CI run for `HEAD` is not green (`gh run list --commit`).
- [ ] Prove the gate catches each failure kind (see Acceptance Criteria) on a throwaway branch.

## Implementation Plan

## Acceptance Criteria

- [ ] `make gate-fast` and `make gate` exist. Their timings on the reference machine are recorded in this story: `gate-fast` ≤ 60 s warm and `gate` ≤ 5 min warm, or the epic's open question is re-decided with the measured numbers.
- [ ] With the hooks installed, a commit with an unformatted file, a clippy warning or a failing test is refused, with the failing command named in the output.
- [ ] A push with a vulnerable or license-incompatible dependency (test with a throwaway branch) is refused by `cargo deny`.
- [ ] A test that sleeps forever fails by timeout instead of hanging the gate.
- [ ] The GitHub Actions run on `main` is green, and a deliberately broken branch shows a red run. The nightly workflow has run at least once.
- [ ] `make dist` runs the full gate first.

## Final Summary

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

---
type: epic
id: "DDC-006"
title: "Code shape gate: measurable conventions that guide humans and agents"
status: planned
priority: p2
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [rust, quality, conventions, tooling, agents]
code_refs: []
review_status: reviewed
reviewed_at: 2026-10-28
---

# DDC-006: Code shape gate: measurable conventions that guide humans and agents

| Field | Value |
|---|---|
| **Status** | planned |
| **Priority** | p2 |
| **Created** | 2026-10-28 |
| **Updated** | 2026-10-28 |

---

## Goal

The code's shape is held by a tool, not by prose. The shape is:
- files small and split by concern;
- short functions with shallow nesting;
- cohesive types;
- modules that depend only in allowed directions.

The rules are numbers in a config file. A checker measures them and reports each violation with the file, line, measured value, limit and the facts needed to fix it, for an agent that has not read a style guide. The report reaches its reader at three moments:
- the agent, right after it edits a file;
- the editor, inline, through rust-analyzer;
- the gate, before a commit.

Success criteria:

- `make gate` fails on any new violation of a shape rule or of the curated clippy lint set. The existing violations sit in a baseline file that can only shrink. Warn-level findings (cohesion, `pub` items) are printed, never baselined. At the end of the epic the Rust baseline is empty; the Swift baseline is listed for a later burn-down.
- A Claude Code session that writes a 120-line function gets the violation back as PostToolUse feedback from `ddc-shape` in the same turn. The hook runs a prebuilt binary, not `cargo`, so it is fast enough to run on every edit.
- rust-analyzer shows shape violations inline next to compiler and clippy diagnostics, with no extra language server.

## Context

Measured on 2026-10-28 (workspace 0.3.0 with the uncommitted `disk/` work, 76 source files, 14,952 raw lines under `src/`). Clippy counts are distinct primary spans: `--all-targets` reports each library warning twice, once per target, and the first count of 904 had that error.

| Measure | Value |
|---|---|
| Source files over 400 code lines (excluding blanks, comments and `#[cfg(test)]` modules) | 2: `scan/walker.rs` 573, `exec/mod.rs` 408 (`rules/catalog.rs`, 602, was split into `rules/catalog/` by DDC-007's story "Rule catalog split: shared and per-OS sections"; every file there is under 400 lines) |
| Source files over 250 code lines | 12 |
| Functions over 100 lines (`clippy::too_many_lines`) | 7 |
| Functions over 70 lines (proposed hard limit) / over 40 (soft) | 11 / 35 |
| clippy `pedantic` + `nursery` warnings | 399 in `src/`, 462 with tests. Top: `must_use_candidate` 64, `missing_errors_doc` 43, `use_self` 41, casts 43 in `src/` |
| `#[allow]` / `#![allow]` sites | 17 (7 in `src/`) |
| Two-module dependency cycles in `ddc-core` | 7, among them `sys↔scan`, `rules↔plan`, `plan↔exec`, `report↔exec`, `store↔disk` |

Today's guardrails:
- `rustfmt.toml`: width 120.
- `clippy.toml`: `disallowed-methods` funnels deletion and `$HOME` access through dedicated modules.
- Workspace lints deny `unsafe_code`, `unsafe_op_in_unsafe_fn`, `dbg_macro` and `disallowed_methods`.
- `ddc-core/src/lib.rs` denies `print_stdout` and `print_stderr`.

Nothing limits size, nesting, complexity, cohesion or module dependencies.

What clippy can and cannot do:
- It has thresholds for function length, nesting (`excessive_nesting`, which also counts `impl` and inline `mod` blocks), cognitive complexity and arguments.
- It has no lint for file length, methods per type, cohesion, or module directions.
- On this machine a warm `cargo clippy -p ddc-core` after one edit takes about 9.5 s, too slow to run on every agent edit.

So the custom checker measures the function-level rules too, cheaply from the syntax tree. Clippy stays the second opinion in the gate and the editor.

"SOLID" and "the best code anyone has seen" are not measurable. This epic measures proxies for them:

| Principle | Proxy |
|---|---|
| Single responsibility | File, function and type size; cohesion of a type's methods (warn-only) |
| Interface segregation | Trait method count |
| Dependency inversion | Allowed directions between modules, no cycles |

Review by a person or by `projectstore:reviewer` remains the judge of design, and the control against gaming the counters.

## Stories

| Story | Status | Description |
|-------|--------|-------------|
| [story-code-shape-conventions-and-the-checkers-design](./stories/story-code-shape-conventions-and-the-checkers-design.md) | planned | ADR: the numeric limits, the module directions with today's cycles, and the checker's architecture. `docs/code-shape.md` gives one line per rule, with its reason |
| [story-clippy-at-full-strength-curated-lints-with-reasons](./stories/story-clippy-at-full-strength-curated-lints-with-reasons.md) | planned | A curated pedantic/nursery/restriction set in workspace lints, thresholds in `clippy.toml`, reasons on every exception, the 399 `src/` warnings fixed or justified |
| [story-ddc-shape-a-syn-based-checker-for-size-nesting-cohesion-and-boundaries](./stories/story-ddc-shape-a-syn-based-checker-for-size-nesting-cohesion-and-boundaries.md) | planned | A workspace tool. It parses every `.rs` file with `syn` and checks file and function size, nesting, parameters, type and trait size, cohesion, module directions and cycles, and which modules may use which external crates. Cargo-compatible output; baseline ratchet; part of the gate |
| [story-agents-and-editors-see-violations-as-they-write](./stories/story-agents-and-editors-see-violations-as-they-write.md) | planned | A Claude Code PostToolUse hook that runs the prebuilt checker on each edited Rust file, a rust-analyzer check override with its own target folder, and a short AGENTS.md pointer |
| [story-burn-down-the-baseline-split-long-files-and-functions](./stories/story-burn-down-the-baseline-split-long-files-and-functions.md) | planned | Behaviour-preserving refactors until the baseline is empty: long files and functions, then the dependency cycles one at a time, the safety path last |
| [story-swiftlint-with-the-same-limits-for-the-app](./stories/story-swiftlint-with-the-same-limits-for-the-app.md) | planned | SwiftLint for `app/` with the ADR's limits translated to its rules, its own baseline, run in the gate when Swift changes |

## Expected Results

- [ ] Limits and module directions live in one config file (`shape.toml`), decided in an ADR.
- [ ] `ddc-shape` reports violations as text and as cargo-compatible JSON.
- [ ] The gate fails on new violations. The baseline is empty at the end.
- [ ] The agent hook and the editor integration work on the reference machine.
- [ ] The curated clippy set is clean, and every exception carries a reason.
- [ ] The Swift app has the same limits through SwiftLint.

## Dependencies

- [[epics/DDC-005/epic|DDC-005 · Production-grade Rust core: mandatory gate and load testing]]: story "Mandatory gate before every commit" provides the gate this epic plugs into. Its golden check (story "Synthetic load trees: large, wide, deep and hostile"), property tests and complexity counters are the safety net for the refactors in the burn-down story.
- [[epics/DDC-007/epic|DDC-007 · Standalone CLI on macOS, Linux and Windows: portable core, release builds and installers]]: its story "Move OS calls behind the platform module" moves `sys/`, `scan/walker.rs`, `exec/*` and `disk/*`, and its story "Rule catalog split: shared and per-OS sections" handles `rules/catalog.rs`. Both land before the burn-down story. The move story also relies on this epic's OS-API rule (OS crates and `std::os::*` only under `platform/`). `ddc-shape` enters the gate after the move, so its starting baseline is measured on moved code.
- `syn` (with the `full` and `visit` features), `proc-macro2` (`span-locations`) and `toml` as dependencies of the tool crate only. The shipped binary does not change.

## Open Questions

- [ ] Should the checker become a reusable tool for other SmartAndPoint Rust repositories (its own repo, a Claude Code plugin)? Proposal: build it inside this workspace first; extract it once a second repository wants it.
- [ ] Do test files get their own limits? Proposal: their own, looser size limits, and the same nesting and complexity limits.
- [ ] Should static data tables (the rule catalog) get a separate, higher size limit rather than an exception? To decide in the ADR. DDC-007's catalog split may make it moot.
- [ ] Committing `.claude/settings.json` with the hook means every clone of the public repository runs it. Proposal: yes, it only reads files and runs the repository's own checker. The owner confirms.

## Related

- [[epics/DDC-005/epic|DDC-005 · Production-grade Rust core: mandatory gate and load testing]]
- [[epics/DDC-007/epic|DDC-007 · Standalone CLI on macOS, Linux and Windows: portable core, release builds and installers]]
- [[adr/rust-core-library-and-cli-with-json-output|Rust core: library + CLI with JSON output]]

---

*Last updated: 2026-10-28*

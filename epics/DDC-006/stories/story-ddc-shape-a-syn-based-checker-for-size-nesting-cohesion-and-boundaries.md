---
type: story
id: "story-ddc-shape-a-syn-based-checker-for-size-nesting-cohesion-and-boundaries"
epic: "DDC-006"
title: "ddc-shape: a syn-based checker for size, nesting, cohesion and boundaries"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-30
external_refs: {}
tags: [rust, tooling, conventions, ast]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# ddc-shape: a syn-based checker for size, nesting, cohesion and boundaries

| Field | Value |
|---|---|
| **Epic** | [DDC-006](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

This story builds `ddc-shape`, a small workspace tool. It parses every `.rs` file with `syn`, with no compiler and on the stable toolchain, measures the ADR's rules, and reports each violation. It measures the function-level rules (length, nesting, parameters) too, although clippy also has them. That way the per-edit agent hook can run this binary alone, in well under a second, without `cargo`.

Each report carries the facts an agent needs and leaves the design decision to the agent or person:

```
crates/ddc-core/src/scan/walker.rs:1: shape::file-length: 573 code lines > 400 (hard).
  Largest items: fn walk_dir L120-L276 (156 lines), impl Walker L300-L520 (14 methods).
  Cohesion components of Walker: {walk_dir, descend, push_entry} / {classify_entry, rule_for, …} (LCOM4 = 2).
```

## Decomposition

- [ ] Crate `tools/ddc-shape` (workspace member, `publish = false`, not shipped). It depends on `syn` (`full`, `visit`), `proc-macro2` (`span-locations`), `toml`, `serde_json`. Add it to `members` and set `default-members = ["crates/ddc-core", "crates/ddc-cli"]`. The Makefile's `--workspace` targets (`clippy`, `test`) include the tool, as intended: it must pass its own rules and the curated clippy set. `make release` switches to `-p ddc-cli` (`scripts/build-dist.sh` already does), so the release build (thin LTO, one codegen unit) does not compile `syn` and the tool.
- [ ] `shape.toml` at the workspace root holds:
  - the limits;
  - the module directions;
  - the external-crate rules;
  - path excludes;
  - exceptions, each with `reason` and `owner`.
- [ ] Measures:
  - code lines per file (comments, blank lines and `#[cfg(test)]` modules counted separately);
  - function body length, nesting depth inside the body, parameter count, cognitive complexity;
  - inherent methods per type, summed across impl blocks in the crate;
  - trait methods;
  - LCOM4 per type with fields, warn-only. Methods are connected when both touch the same field through `self.x` or a `Self { x, .. }` pattern, or when one calls the other through `self.f()` or `Self::f(`. Constructors, trivial accessors, trait impls, unit structs and field-less enums are skipped.
  - `pub` items per module, warn-only;
  - module directions and cycles, from the graph source the ADR chose (`cargo modules` DOT, or its own `use` graph with `super::`/`self::` resolution and its stated limits). Any strongly connected component of more than one module fails the cycle rule, not only two-module cycles;
  - OS APIs per module, a path-prefix rule per crate: `libc::`, `rustix::`, `windows_sys::`, `windows::` and `std::os::` (any submodule) only under `ddc-core/src/platform/`, `ddc-cli/src/platform/` and `ddc-loadgen/src/platform/` (the rule of AC3 in DDC-007's story "Move OS calls behind the platform module", which this rule replaces; when they differ, that AC wins). A `cfg`, `cfg!` or `cfg_attr` on `target_os`, `target_family`, `unix` or `windows` outside those folders is a separate attribute rule.
- [ ] Messages: for each rule, the measured value, the limit and the facts that help (the largest items for file length; the method groups for cohesion; the `use` line and the allowed direction for a layer violation). No invented destinations.
- [ ] Output: human text (default), and `--format cargo`, cargo-compatible `compiler-message` JSON lines that rust-analyzer and the hook read.
- [ ] Baseline `shape-baseline.json`, keyed by (rule, crate, item path such as `ddc_core::scan::walker::walk_dir`), not by file path. It fails on a new violation or a baselined one that got worse, and prints "baseline can shrink" when one got better. `--update-baseline` only shrinks it. A move is defined: a baselined key that disappears, plus a new finding with the same (rule, crate, item kind, last path segment) and a value ≤ the old one. `--update-baseline` rewrites the key and prints `moved a -> b`. Anything else is a new violation. The baseline's entry count never grows.
- [ ] `--files <paths>` checks single files: the file- and function-level rules only. When it reports violations, it adds one line saying that crate-level rules (directions, methods summed across files) were skipped and run in the gate. A clean file prints nothing.
- [ ] `make shape` builds `target/release/ddc-shape` if needed and runs it on the workspace. `make tools` and `make hooks` build it too, so the agent hook finds it. It joins the fast gate (target ≤ 2 s).
- [ ] Tests: fixture files for each rule (passing and failing), a golden test of each output format, and a test that the baseline only shrinks or moves.

## Implementation Plan

## Acceptance Criteria

- [ ] `make shape` runs on the workspace in ≤ 2 s warm and is part of `make gate-fast`.
- [ ] Every rule in `docs/code-shape.md` has a fixture test in `ddc-shape`.
- [ ] `--format cargo` output is shown by rust-analyzer as diagnostics (evidence in story "Agents and editors see violations as they write").
- [ ] A new 450-code-line file fails the gate with a message naming the file, its length, the limit and its largest items; a baselined item that grows fails too; a baselined item that moves to another file with the same value does not.
- [ ] A `use libc` and a `use std::os::unix::fs::MetadataExt` outside `platform/` each fail the OS-API rule (fixtures). A three-module cycle fails the cycle rule (fixture). A module rename that moves a baselined function passes and prints the move (fixture).
- [ ] `ddc-shape` passes its own rules.

## Final Summary

## Technical Notes

- `syn` sees macro invocations as token streams, so function bodies produced by `macro_rules!` are not measured. Record that as a known limit in `docs/code-shape.md`.
- Spans need `proc-macro2`'s `span-locations` feature for line numbers outside a proc-macro context.
- Resolve modules through `mod` declarations, never by guessing from file paths: `#[path]` exists.

## Dependencies

- Story "Code shape conventions and the checker's design".
- DDC-005 story "Mandatory gate before every commit".
- DDC-007 story "Move OS calls behind the platform module": the tool and its fixtures can be built before it, but `make shape` joins `make gate-fast` and the baseline is first recorded only after the move (epic Dependencies).

## Attachments

-

---

*Last updated: 2026-10-30*

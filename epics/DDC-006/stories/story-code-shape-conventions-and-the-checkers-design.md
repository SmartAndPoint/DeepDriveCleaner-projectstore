---
type: story
id: "story-code-shape-conventions-and-the-checkers-design"
epic: "DDC-006"
title: "Code shape conventions and the checker's design"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [rust, conventions, adr, tooling]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Code shape conventions and the checker's design

| Field | Value |
|---|---|
| **Epic** | [DDC-006](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

Before any tool is built, two things are decided and recorded in an ADR. The ADR goes through the critic.
- **What** the rules are, as numbers and as allowed directions between modules.
- **How** they are checked.

`docs/code-shape.md` then lists each rule in one line with its reason, for people. The tool's messages carry the same reason, for agents.

Proposed limits, to be confirmed or changed in the ADR after looking at the distribution, not just the counts:

| Rule | Soft (warn) | Hard (fail) | Applies to |
|---|---|---|---|
| File length: code lines, excluding blank lines, comments and `#[cfg(test)]` modules | 250 | 400 | `src/` |
| File length, test files and `#[cfg(test)]` modules | 400 | 700 | tests |
| Function body length, in code lines as clippy's `too_many_lines` counts them (no blank or comment lines) | 40 | 70 | all, including functions in inline `#[cfg(test)]` modules |
| Nesting inside a function body (the body itself is 0; each braced block, match-arm block or closure body adds 1) | 3 | 4 | all |
| Cognitive complexity, by clippy's algorithm so both tools agree (the ADR states it) | 10 | 15 | all; measure first, since clippy's lint at 25 fires on nothing today |
| Parameters | 5 | 6 | all |
| Inherent methods per type, summed across `impl` blocks | 12 | 20 | `src/` |
| Trait methods | 6 | 9 | `src/` |
| Cohesion: LCOM4 over inherent methods, excluding constructors, trivial accessors and trait impls | 2 | warn only | `src/` |
| `pub` items per file module (unit to be fixed in the ADR) | measure | warn only in v1 | `ddc-core` |

Today's dependency graph of `ddc-core` (`crate::` paths, 2026-10-28) has 7 two-module cycles, and 3-module cycles through `exec`, `plan`, `report` and `rules` that fall with them:
- `sys↔scan` (`sys/bulk.rs` uses `scan::reader`);
- `model↔rules`;
- `rules↔plan` (`rules/listing.rs` uses `plan::action_for`);
- `plan↔exec` (`plan.rs` uses `exec::SkipReason`);
- `report↔exec` (`report/after_clean.rs`);
- `report↔plan` (`report/recheck.rs`);
- `store↔disk`.

`disk`, `store`, `settings`, `notify`, `progress` and `env` need a place in the layering too. The ADR decides which edges are design intent (for example, shared types in `model` used downward) and which are violations. Layer rules enter the gate as warn-only, until the burn-down story has a plan for each cycle.

Options for the checker, to be weighed in the ADR:

| Option | For | Against |
|---|---|---|
| clippy thresholds only | built in, zero code | no file length, type size, cohesion or directions; ~9.5 s per crate, too slow for a per-edit hook |
| `syn`-based tool in the workspace (proposed) | exact syntax tree, stable toolchain, fast (a parse of 15k lines takes tens of milliseconds), any rule we can state, own messages | we maintain it; macro bodies are opaque; no name resolution |
| `cargo modules` (`dependencies --acyclic`, DOT output) | resolved paths through rust-analyzer; cycles for free | no layer directions; an extra tool to install; DOT only |
| Dylint (custom lint libraries) | real lints with type information | nightly toolchain and dylib plumbing for every developer and CI |
| `rust-code-analysis` metrics | ready metrics (cyclomatic, cognitive, Halstead) | metrics only; maintenance to check |
| A custom LSP server | live in every editor | heavy; rust-analyzer's `check.overrideCommand` gives the same inline result from a command that prints cargo JSON |
| Shell or Python line counting | trivial | no syntax tree: wrong on strings, macros and nested items |

The proposal:
- `ddc-shape` (syn) measures every rule, including the function-level ones clippy also has, so the agent hook never needs `cargo`.
- Clippy remains the second opinion in the gate and the editor.
- For the module graph, the ADR chooses between `ddc-shape` reading `cargo modules`' DOT output (resolved, with directions checked on top) and its own `use` graph (no extra tool, with its resolution limits written down).

## Decomposition

- [ ] Measure today's code against the proposed limits and record the distribution per rule, including `.unwrap(`/`.expect(` sites in `src/` (about 114 today, some in inline test modules) for the clippy story. Count clippy findings by distinct primary span: `--all-targets` reports library warnings twice. Run clippy with `PATH=$HOME/.cargo/bin:$PATH`, because Homebrew's clippy 1.81 shadows it on the reference machine and silently reports nothing.
- [ ] Draw the module dependency graph of `ddc-core` and `ddc-cli`. Write the directions as intended, and list each current cycle with its decision.
- [ ] Decide the treatment of static data tables (catalog): their own limit, an exception, or the DDC-007 split.
- [ ] ADR "Code shape: limits, layers and the checker". Read `adr/README.md` first. It records:
  - the limits;
  - the directions;
  - the options above with the decision;
  - the baseline-ratchet policy;
  - how an exception is written (`#[expect]` with a reason, or a `shape.toml` entry with a reason and an owner).
- [ ] `projectstore:critic` on the ADR; revise until accepted.
- [ ] `docs/code-shape.md`: one line per rule, its limit and why, and the known limits of the checker (macros, name resolution). It links the ADR and does not restate it.

## Implementation Plan

## Acceptance Criteria

- [ ] The ADR exists in `adr/`, critic-approved, with status accepted.
- [ ] `docs/code-shape.md` lists every rule from the ADR with its limit, and nothing the tools will not check.
- [ ] The measured distribution per rule and the list of cycles with a decision for each are recorded in this story (evidence for the burn-down size).

## Final Summary

## Technical Notes

- If clippy's `excessive_nesting` is kept as a second opinion, its count includes the `impl` block, inline `mod` blocks and the function body. Set its threshold to the hard limit plus 2, and note that code in an inline test module gets one level less.

## Dependencies

- None. This story comes first in the epic.

## Attachments

-

---

*Last updated: 2026-10-28*

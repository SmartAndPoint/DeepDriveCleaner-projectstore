---
type: story
id: "story-swiftlint-with-the-same-limits-for-the-app"
epic: "DDC-006"
title: "SwiftLint with the same limits for the app"
status: planned
priority: p3
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [swift, app, lints, conventions]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# SwiftLint with the same limits for the app

| Field | Value |
|---|---|
| **Epic** | [DDC-006](../epic.md) |
| **Status** | planned |
| **Priority** | p3 |
| **Assignee** | Evgenii Konev |

---

## Description

The menu bar app has about 15,000 lines of Swift under `app/`, with no linter. The largest source file, `DDCKit/Sources/DDCKit/PopoverModel.swift`, has 625 lines (the largest test file has 738). The owner's wish for small, clean code applies to all of the product, not only the Rust core.

SwiftLint ships close matches, so no custom tool is needed. The differences are stated, not hidden:
- `file_length` counts raw lines; with `ignore_comment_only_lines: true` blank lines still count, so its number is not the ADR's code lines;
- `function_body_length` ignores comments and blank lines, matching the ADR;
- `function_parameter_count` matches;
- `cyclomatic_complexity` is the proxy for block nesting and cognitive complexity: SwiftLint has no block-nesting rule;
- `nesting` limits nested type and function declarations only; it stays at its defaults;
- `type_body_length` has no Rust counterpart (the ADR limits methods per type); the ADR gives it a Swift-only number.

This story sets them to the ADR's limits, adds a baseline, and runs SwiftLint in the gate when Swift files change.

## Decomposition

- [ ] `.swiftlint.yml` at `app/` with the ADR's limits translated as above, plus `force_unwrapping` and `force_try` outside tests.
- [ ] A baseline (`swiftlint --write-baseline`) for today's violations; the gate uses `--baseline` and fails only on new ones.
- [ ] SwiftLint pinned (Homebrew version or the Swift package plugin) and installed by `make tools`.
- [ ] `make swiftlint`, in the fast gate when `git diff --cached` touches `app/**/*.swift`, and in DDC-005's gate workflow (macOS runner).
- [ ] The PostToolUse hook of story "Agents and editors see violations as they write" also runs `swiftlint lint <file>` (check the argument form against the pinned version) for edited Swift files, if it fits the 1 s budget.
- [ ] A follow-up list in this story: the files over the hard limits, for a later burn-down.

## Implementation Plan

## Acceptance Criteria

- [ ] `make swiftlint` passes on today's code with the baseline, and fails on a new 500-line Swift file (test on a throwaway branch).
- [ ] The gate and CI run it when Swift changes.
- [ ] The limits in `.swiftlint.yml` match the ADR rule by rule, with the Swift-only differences listed in `docs/code-shape.md`.

## Final Summary

## Technical Notes

- Generated Xcode files and build output are excluded by path.

## Dependencies

- Story "Code shape conventions and the checker's design".
- DDC-005 story "Mandatory gate before every commit".

## Attachments

-

---

*Last updated: 2026-10-28*

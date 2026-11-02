---
type: story
id: "story-swiftpm-build-folders-as-rebuildable-artifacts"
epic: "DDC-001"
title: "SwiftPM build folders as rebuildable artifacts"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-11-02
updated: 2026-11-02
external_refs: {}
tags: [rules, swift, worktree]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# SwiftPM build folders as rebuildable artifacts

| Field | Value |
|---|---|
| **Epic** | [DDC-001](../epic.md) |
| **Status** | planned |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

A Swift package's build folder, `.build` next to `Package.swift`, holds everything `swift build` makes: the compiled modules, the dependencies' checkouts and the build database. `swift build` makes it again. ddc has no rule for it: the scan does not offer it, and inside a git worktree it is not an artifact that accounts for ignored content.

The reference-machine check of [[epics/DDC-003/stories/story-worktree-findings-and-provenance|Worktree findings and provenance]] (DDC-003, recorded in its final summary) found this on 2026-11-02: four Claude Code worktrees stayed `review` for one reason only, a package's `.build` reported as "ignored content no artifact accounts for". Each was clean and idle for days. With a rule, the `.build` folder is a finding of its own and an artifact that accounts for itself, so condition 5 holds and those worktrees go on to condition 6 like any other.

The package there declares no dependencies, so it has no `Package.resolved`, and SwiftPM never writes one for such a package. The rule therefore has no lock-file gate, like `rust.target`, `gradle.build` and `cmake.build`: a build output is rebuildable whether or not a lock file pins its inputs.

## Decomposition

- [ ] `swift.build` in `rules/catalog/shared/projects.rs`: a directory named `.build` whose parent holds `Package.swift` (`Matcher::Name` with a sibling, as `rust.target`), category project artifact, class `rebuildable`, removal `delete`, no lock files; hint "rebuilt by swift build; dependencies are fetched again"
- [ ] Registered in `shared/mod.rs` (`all()`, `SHARED_IDS`, `COUNT` 77 → 78) and placed in `macos.rs::RULES` and `linux.rs::RULES` before `shared::CMAKE_BUILD`, at the same position in both (the section test requires the macOS order on every platform), since `.build` is one of `Matcher::CmakeBuild`'s conventional names; `macos.rs` `COUNT` 93 → 94; `windows.rs` takes it from `shared::all`, and `linux.rs` and `windows.rs` derive their counts
- [ ] `rules/listing.rs` `GROUPS`: in "Build outputs"
- [ ] `tests/common/fixtures.rs` `rule_fixture`: a fixture for it
- [ ] `bench/make_baseline.py`: a `swift.build` branch before its CMake conventional-name check; `make golden-record`, with a golden diff that follows only from this rule
- [ ] Tests (below); new test names in `.config/native-skips/windows.txt`, where the scanner is not ported
- [ ] README's class table: `.build` (Swift packages) among the `rebuildable` examples
- [ ] An addendum row in [[epics/DDC-001/stories/story-detection-rule-catalog|Detection rule catalog]], whose rule table and "hidden directory rule" paragraph list every rebuildable project rule

## Implementation Plan

<!-- Written at the work-start gate (/projectstore:story plan), AFTER studying the
     codebase. When a spec covers this story, this is a thin route through the
     spec's contracts: which contracts, in what order, which files. -->

## Acceptance Criteria

- [ ] Test: `<proj>/.build` with `<proj>/Package.swift` is a `swift.build` finding, `rebuildable`, with and without `<proj>/Package.resolved`, selected once the project is idle for `--min-age` days
- [ ] Test: a `.build` with no `Package.swift` beside it is not a `swift.build` finding; a `.build` that is a symlink is not a finding; a `.build` holding `CMakeCache.txt` beside a `Package.swift` is `swift.build` (the order before `cmake.build`), and the same folder without `Package.swift` stays `cmake.build` (a `CMakeCache.txt` as the `cmake.build` fixture writes it)
- [ ] Test (worktrees, real git): a clean Claude Code worktree whose only ignored content is a package's `.build` (`Package.swift` tracked, no `Package.resolved`, `.build` ignored) is eligible and selected, and its `.build` is its own finding inside it
- [ ] `ddc rules --json` lists `swift.build` in group "Build outputs" with class `rebuildable`, method `delete` and its hint, on macOS, Linux and Windows (`--platform`)
- [ ] `make gate-fast` passes, and the Windows leg (`scripts/native-windows.sh`) is green with the new skips listed

## Final Summary

<!-- Written at the done gate (/projectstore:story close): what changed, why,
     tests executed, risks and follow-ups. -->

## Technical Notes

- Xcode builds packages into DerivedData, which `xcode.derived-data` covers; this story is SwiftPM's in-tree `.build` alone.
- `.build` is also a CMake conventional build folder name (`Matcher::CmakeBuild`, `bench/make_baseline.py`); the `Package.swift` sibling is what makes it SwiftPM's, hence the order.
- A dependency's own `Package.swift` inside `.build/checkouts/` lies inside the claimed `.build`, and claims do not nest, so there is no second finding.
- Not in this story, follow-ups: SwiftPM's global cache (`~/Library/Caches/org.swift.swiftpm` on macOS, `~/.cache/org.swift.swiftpm` on Linux), the analog of `cache.cargo`; `.swiftpm/` (Xcode's workspace and schemes for an opened package), which will hold a worktree back the same way; a build folder moved elsewhere with `--scratch-path`. `~/Library/org.swift.swiftpm` is configuration, not a cache, and is never a finding.

## Dependencies

- [[epics/DDC-003/stories/story-worktree-findings-and-provenance|Worktree findings and provenance]] (DDC-003): the ignored-content check counts every `safe` or `rebuildable` artifact removed with `delete` (`rules/worktree.rs`, `removable_owners`), by its effective class, so a `swift.build` finding accounts for its folder with no change there
- [[epics/DDC-001/stories/story-detection-rule-catalog|Detection rule catalog]] (DDC-001): the rule table this rule extends

## Attachments

-

---

*Last updated: 2026-11-02*

---
type: story
id: "story-rule-catalog-split-shared-and-per-os-sections"
epic: "DDC-007"
title: "Rule catalog split: shared and per-OS sections"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-30
external_refs: {}
tags: [rust, rules, portability]
code_refs: ["crates/ddc-core/src/rules/catalog/", "crates/ddc-core/src/rules/listing.rs", "crates/ddc-core/src/rules/mod.rs", "crates/ddc-core/src/rules/toolchain.rs", "crates/ddc-core/src/exec/guard.rs", "crates/ddc-core/src/exec/preflight.rs", "crates/ddc-core/src/report/contents.rs", "crates/ddc-cli/src/cli.rs", "crates/ddc-cli/src/cmd/rules.rs", "crates/ddc-cli/tests/rules_cli.rs", "crates/ddc-cli/tests/golden_cli.rs", "crates/ddc-core/tests/rules.rs", "crates/ddc-core/tests/matcher_reference.rs", "crates/ddc-loadgen/src/fixtures.rs", "crates/ddc-loadgen/src/manifest.rs", "crates/ddc-loadgen/src/items.rs", "crates/ddc-loadgen/tests/properties.rs", "README.md", "app/Packages/DDCKit/Sources/DDCKit/SkipSummary.swift", "app/Packages/DDCKit/Tests/DDCKitTests/ContractTests.swift"]
specs: []
started_at: "2026-10-30T13:29:14.929Z"
closed_at: "2026-10-30T15:12:30.015Z"
plan_updated_at: "2026-10-30T13:29:14.929Z"
---

# Rule catalog split: shared and per-OS sections

| Field | Value |
|---|---|
| **Epic** | [DDC-007](../epic.md) |
| **Status** | done |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

`rules/catalog.rs` (631 lines) mixes rules that hold everywhere with rules that name macOS paths:
- everywhere: `node_modules`, `target`, `.venv`, `~/.cargo/registry`, `~/.npm`, `~/.gradle`, `~/.m2`;
- macOS only: `~/Library/Caches/…`, Xcode, simulators.

The protected-folder list of the safety ADR §6 is also macOS-shaped. This story splits both into a shared section and per-OS sections, so the Linux and Windows stories add rules without touching macOS. The macOS catalog and its output do not change.

## Decomposition

- [x] Classify every rule: shared (same relative path everywhere), shared with per-OS paths (one rule id, a path per OS, e.g. pip's cache), or OS-only.
- [x] The catalog becomes `rules/catalog/{shared,macos,linux,windows}.rs`. Linux and Windows start empty except the shared rules. Rule ids are unchanged, so settings that disable rules keep working.
- [x] The protected-folder list splits the same way. The safety ADR §6 list stays the macOS list.
- [x] `ddc rules` prints the rules of the running platform, and `--platform <os>` prints another platform's, for review.
- [x] Golden test: `ddc rules --json` and `scan --json` on all S profiles are identical on macOS before and after.

## Implementation Plan

The how follows ADR [[adr/platform-seam-one-platform-module-per-os-with-declared-capabilities|Platform seam: one platform module per OS with declared capabilities]] (§4 and §8, amended 2026-10-30 by this story) and §6 of [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]. It also follows the pattern of the closed story "Move OS calls behind the platform module": one file per OS, shared code never names an OS, and no `cfg` outside `platform/`. Each step below is its own commit and passes `make gate`. `make golden-check` stays unchanged, and this story never runs `golden-record`.

Decided 2026-10-30, from the planner's findings:

- **No JSON change.** `rules --json` gains no `platform` field and keeps schema version 1. The seam ADR §4 had put that field and a schema bump in this story. That would change `bench/golden/rules.json` and break the app, which refuses any rules schema but 1 (`RuleList.swift`). AC1 asks for golden-equal output, and a new field fails the golden comparison. A reader of `--platform` already knows which platform it asked for. The field and the bump go with story "Linux scanner and rule catalog", whose flip of `SUPPORTED` is the first time two platforms print different real catalogs. The ADR is amended.
- **Per-OS paths in the human output only.** `RuleInfo` carries no paths, and adding them would change the macOS JSON. AC3 is met by the human listing: every human `ddc rules` output, plain or with `--platform`, prints each rule's home-anchored paths as `~/…` lines. No test or golden pins the macOS human text.
- **The section is chosen at run time.** `rules_of(p: Platform)` matches on a `Platform` value: `ddc_core::platform::PLATFORM` for the running OS, or the `--platform` argument. No `cfg` appears in `rules/`, so AC3 of the move story still holds.
- **One rule id, one definition.** A rule shared with per-OS paths is a `const fn x(p: Platform) -> Rule` in `shared.rs` that picks its paths from a `PerOs { macos, linux, windows }` table. Each OS list resolves it to a plain `Rule`. `Rule`, `Matcher`, `Catalog`, the matcher and the visitor do not change.
- **Order is the macOS order.** The list index is a rule's tag, the first specialized match wins, and `rules --json` prints in list order. `macos.rs` lists all 77 rules in today's order. `linux.rs` and `windows.rs` take that order filtered to the shared rules. Every list holds `user.kept`.
- **Lists are `static`.** `Catalog::tag_of` compares with `ptr::eq`, so a `const` list copied at each use would break it. `RULES` and `rules_of(PLATFORM)` are the same slice.
- **The `windows::` trap.** The move story's structure test flags the text `windows::` outside `platform/`. The file stays `windows.rs`, and `catalog/mod.rs` writes `mod windows; use self::windows as win;`, with a comment. The test is not loosened.
- **Protected folders split the same way.** Shared in home: `""`, `Documents`, `Desktop`, `Pictures`, `Music`, `Downloads`, `Projects`, `.config`, `.ssh`, `.gnupg`, `.local`, `.cache`. macOS in home: `Movies`, `Library` and its six protected subfolders. System folders are per OS: macOS keeps today's, and Linux and Windows start empty. That is safe while `SUPPORTED` is false: `require_supported` refuses every delete. The safety ADR §6 list stays the macOS list (shared plus macOS), by amendment.
- **Classification of the 77 rules:** 38 shared, 10 shared with per-OS paths and 29 macOS-only, as in the planner's table.
  - Shared with per-OS paths: `cache.npm`, `cache.yarn`, `cache.pnpm`, `cache.uv`, `cache.pip`, `cache.poetry`, `cache.go-build`, `browsers.playwright`, `browsers.cypress` and `user.movies` (`Movies`, `Videos`, `Videos`).
  - macOS-only: `macos.app-bundle`; Xcode; `cache.homebrew` and `cache.cocoapods`; Docker Desktop, OrbStack, Rancher Desktop and `cache.colima`; every IDE and app cache rule; `trash`; and the iOS, iMazing, Telegram, WhatsApp and cloud user-data rules.
  - A rule is shared with per-OS paths only where the tool's own documentation gives its default location on all three OSes; the `const fn` cites it in a comment. Everything else stays macOS-only until the Linux or Windows research note promotes it.
  - A plain shared rule is one whose relative path the tool uses on every OS where it runs. A path that never exists on an OS costs nothing, so the shared set means "paths that could exist": `vm.lima` and `toolchain.nvm` are shared though neither runs natively on Windows.
  - Every rule with `owners` is macOS-only: owners are macOS app bundle names.
  - The Windows paths are home-relative (`AppData/Local/…`) placeholders. ADR §8 already allowlists `LOCALAPPDATA`, and deferred `APPDATA` and `TEMP` "to the catalog split or Windows story": this story hands both to the Windows story, which also replaces the placeholders with paths under `%LOCALAPPDATA%`.

Steps:

1. **Pure move.**
   - `rules/catalog.rs` becomes `rules/catalog/{mod,macos}.rs`. The builders, the shared helper lists and the Docker and conda constants go to `mod.rs`. The 77-rule list goes to `macos.rs`.
   - `RULES` is still the macOS list on every platform, as today.
   - The Swift contract test's path (`ContractTests.swift`, `dockerDetails`) and the `SkipSummary.swift` doc comment point at `catalog/mod.rs`.
   - Golden unchanged, and `make ddckit-test` passes (the gate does not run it).
2. **Shared section.**
   - `shared.rs` holds the shared rules and the per-OS `const fn`s with `PerOs`. `macos.rs` refers to them in place, in the same order. `linux.rs` and `windows.rs` (aliased `win`) list the shared rules.
   - `rules_of(p)` is added, and `RULES` follows `PLATFORM`.
   - `RuleList::for_platform(p)` is added here, not in step 4, with its groups filtered to those with a rule on `p` (all 17 on macOS). The tests that pin macOS catalog facts then name `Platform::Macos` in this same commit, so the native Linux and Windows CI jobs stay as green as before: `tests/rules.rs` `every_rule_has_a_group`, `rules/toolchain.rs`'s sibling check, `rules_cli.rs` `rules_json_groups_follow_the_settings_types`, and `golden_cli.rs` `golden_rules`. `golden_rules` compares `rules --platform macos --json` with `rules.json`, so step 2 adds the bare `--platform` argument (a `PlatformArg` value enum in `cli.rs`, converted into `Platform`) and the README synopsis, because `readme_cli.rs` checks that the guide names every option of `--help`. Step 4 adds its human output. Also check that no ungated loadgen property test indexes `CLOUD`, which becomes empty on Linux (`ddc-loadgen/tests/properties.rs`). `make gate` covers macOS only; the native jobs are checked from the PR's CI run.
   - Add the AC2 test. On each platform the ids are unique; `ids(linux) == ids(windows)`; `ids(linux)` is an order-preserving subsequence of `ids(macos)`; every id on more than one platform is defined in `shared.rs`, observed through an id list that `shared.rs` exports (`ids(linux) ⊆ SHARED_IDS ⊆ ids(macos)`); and for each per-OS rule the three resolved `Rule`s agree on every field except the matcher's paths.
   - Diff the old and new macOS id sequences in review.
3. **Protected folders by section.** `protected_in_home(p)` and `protected_system(p)` replace the two constants. All six consumers change: `Catalog.protected` in `rules/mod.rs`, `exec/guard.rs`, `exec/preflight.rs`, `report/contents.rs`, `ddc-core/tests/matcher_reference.rs`, and `ddc-loadgen/tests/properties.rs` (another crate). Tests:
   - the macOS set equals the §6 list, and no entry sits in two sections;
   - **the flip guard**, which runs on every CI OS: `SUPPORTED` may be true only where `protected_system(PLATFORM)` holds the filesystem root (`/`, or a drive root on Windows) and `protected_in_home(PLATFORM)` holds more than the shared section. This test, not prose, stops a platform from being switched on with empty system folders. It binds once stage 2 of story "Build matrix: every target built, tested and packaged from one tag" makes the native CI jobs required, which is a dependency of the Linux story; until then those jobs are continue-on-error.
4. **`ddc rules --platform <os>`.**
   - Paths under each rule in the human output.
   - Tests in `rules_cli.rs`:
     - Linux and Windows list every shared id and no `xcode.` id;
     - `cache.pip` shows `~/.cache/pip` on Linux, and its Windows path differs from the Linux one (no Windows path is pinned, so the Windows story's real paths do not break it);
     - `--platform linux --json` holds the contract;
     - `--platform nope` exits 2.
   - `golden_rules` also compares plain `rules --json` with `rules.json` when `PLATFORM` is macOS (a runtime `if`).
5. `projectstore:reviewer`, then the vault:
   - code_refs: every artifact whose code_refs names `rules/catalog.rs` is repointed to `rules/catalog/` (prose mentions in closed stories stay as history), and DDC-006's size table row for `catalog.rs` is updated;
   - the Linux and Windows stories' system protected folders (below).

Risks, each with a guard:
- **Order or tag drift.** Guard: `golden-check` and the id-sequence diff.
- **Pointer identity.** Guard: lists are `static`, and a test asserts `ptr::eq(RULES, rules_of(PLATFORM))`.
- **Size.** `shared.rs` may pass DDC-006's 400-line limit. If it does, split it into a `shared/` folder by category; the shape gate is not enforced yet.
- **`ENTRY_RULES` names `user.ios-backups`,** which Linux and Windows lack. It is only checked with `contains`, so this is harmless.

## Acceptance Criteria

- [x] On macOS, `ddc rules --json` and the S-profile scans are golden-equal before and after: `make golden-check` (DDC-005 story "Synthetic load trees: large, wide, deep and hostile") passes with the same rule ids in the same order and the same fields, and `make golden-record` is never run. — evidence: `golden_cli.rs` `golden_rules` (`rules --platform macos --json`, and plain `rules --json` on macOS, against `bench/golden/rules.json`); `make golden-check` 13/13 on the final tree; `git diff 72c3765 -- bench/golden` empty; the reviewer's dump of every rule's `Debug` output and both protected lists, before and after, is identical (77 rules, same order)
- [x] Every rule id appears in exactly one section, or in the shared section with per-OS paths (test, as defined in step 2). — evidence: `rules/catalog/mod.rs` `every_rule_id_is_in_one_section_or_shared` (unique ids; linux == windows; linux an order-preserving subsequence of macos; `ids(linux) ⊆ SHARED_IDS ⊆ ids(macos)`; each entry Debug-equal to `shared::all(p)`; per-OS rules differ only in matcher paths) and `rules_is_the_list_of_this_platform`
- [x] `ddc rules --platform linux` and `--platform windows` list the shared rules with their per-OS paths (in the human output; the JSON is unchanged, see the plan). — evidence: `rules_cli.rs` `every_platform_lists_the_shared_rules_and_only_its_own`, `the_text_shows_each_platforms_paths`, `another_platforms_json_holds_the_contract`, `an_unknown_platform_is_a_usage_error`; `readme_cli` names `--platform`
- [x] The protected folders are split into a shared section and per-OS sections; the macOS set equals the safety ADR §6 list, and the flip guard of step 3 holds (tests). — evidence: `rules/catalog/mod.rs` `the_macos_protected_folders_are_the_safety_adrs_list` (item by item against §6, no folder in two sections) and the flip-guard test (`may_be_supported(Macos)`, `!may_be_supported(Linux|Windows)` on every OS, and it reads each platform module's `SUPPORTED` line, so the required macOS job fails a flip without system folders); all six consumers call `protected_in_home(PLATFORM)` / `protected_system(PLATFORM)`

## Final Summary

**What changed.** `rules/catalog.rs` became `rules/catalog/`: `macos.rs` (77 rules, order unchanged), `linux.rs` and `windows.rs` (48 each, built by `shared::all(p)`), and a `shared/` folder by category (`caches`, `projects`, `tools`, `user`) whose per-OS paths come from `const fn`s. `RULES` is `rules_of(PLATFORM)`; the lists are named statics because `Catalog::tag_of` finds a rule by address. The protected folders split the same way (`protected_in_home(p)`, `protected_system(p)`), with Linux and Windows holding only the shared section until their stories. `ddc rules --platform macos|linux|windows` prints another platform's catalog, and every human listing prints each rule's home paths; `rules --json` is unchanged (schema 1). The load generator names the macOS catalog (`fixtures::catalog()`), so the S trees, their pinned hashes and `bench/golden` describe macOS on every OS.

**Why.** The Linux and Windows catalog stories add rules to their own sections without touching macOS, and the flip guard keeps either from being switched on before its system folders exist.

**Tests executed.**
- `make gate` green on the final tree (626/626 Rust tests, the Python suites), `make cross-check` (Linux gnu and musl, Windows msvc), `make golden-check` 13/13, `make ddckit-test` 379 tests. Code PRs #8 (the story) and #10 (follow-up) merged; Gate run 41082747167 on main green.
- Native CI, compared with the run on PR #4 before the split (`--no-fail-fast`): Linux 36 failures against 37 before, none new; Windows 59 against 59, one new — `rules.rs` `verify_uses_the_same_matching`, which built the running platform's catalog and expected `~/.cache/uv`. PR #10 pins it to the macOS catalog. `names_that_differ_only_in_case_are_one_folder` now skips where walking is unsupported.
- The reviewer simulated Linux on macOS (`PLATFORM` Linux, `SUPPORTED` false) and caught four loadgen proptests that built an empty `select` before their skip and the S-hash pin reading the platform's catalog; both were fixed before merge and the simulation then matched the pre-story baseline.

**Risks and follow-ups.**
- The remaining native failures (36 Linux, 58 Windows) predate this story; they are the work of stage 2 of story "Build matrix: every target built, tested and packaged from one tag", which makes the native jobs required. Until then the flip guard binds through its macOS-job half.
- Windows paths in `shared/caches.rs` are documented placeholders; story "Windows scanner and rule catalog" replaces them and decides `APPDATA`/`TEMP` and an `Env`-based protected set.
- `user.movies` resolves to `~/Videos` on Linux and Windows, which no section protects yet; the Linux and Windows stories add their own protected folders.

## Technical Notes

- Data and log folders per OS, which the epic's table once gave this story, are already done: `platform::folders::default_data_and_log` (story "Move OS calls behind the platform module").
- `bench/make_baseline.py` keeps its own table of macOS catalog paths and ids. This story leaves macOS unchanged, so it is unaffected; a per-OS baseline would diverge there.
- This also helps DDC-006: `catalog.rs` (602 code lines) is one of the three files over DDC-006's 400-line limit, and the split by platform and category removes it from the shape baseline.

## Dependencies

- Story "Move OS calls behind the platform module".

## Attachments

-

---

*Last updated: 2026-10-30*

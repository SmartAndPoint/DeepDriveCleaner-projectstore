---
type: story
id: "story-platform-seam-adr-and-capability-model"
epic: "DDC-007"
title: "Platform seam: ADR and capability model"
status: done
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-30
external_refs: {}
tags: [rust, architecture, portability, adr, schema]
code_refs: ["crates/ddc-core/src/platform/", "crates/ddc-core/src/refusal.rs", "crates/ddc-core/src/model.rs", "crates/ddc-core/src/report/mod.rs", "crates/ddc-core/src/scan/walker.rs", "crates/ddc-core/src/scan/mod.rs", "crates/ddc-core/src/disk/", "crates/ddc-core/src/exec/mod.rs", "crates/ddc-core/src/exec/preflight.rs", "crates/ddc-core/src/exec/report.rs", "crates/ddc-core/src/free/", "crates/ddc-core/src/lib.rs", "crates/ddc-core/Cargo.toml", "crates/ddc-core/tests/unported.rs", "crates/ddc-core/tests/platform.rs", "crates/ddc-core/tests/common/mod.rs", "crates/ddc-cli/src/gate.rs", "crates/ddc-cli/src/main.rs", "crates/ddc-cli/src/cli.rs", "crates/ddc-cli/src/exit.rs", "crates/ddc-cli/src/cmd/disk.rs", "crates/ddc-cli/src/cmd/disk_free.rs", "crates/ddc-cli/Cargo.toml", "crates/ddc-cli/tests/unported_cli.rs", "crates/ddc-cli/tests/structure.rs", "docs/error.schema.json", "docs/scan.schema.json", "docs/clean.schema.json", "docs/disk.schema.json", "app/Packages/DDCKit/Sources/DDCKit/Platform.swift", "app/Packages/DDCKit/Sources/DDCKit/ScanReport.swift", "app/Packages/DDCKit/Sources/DDCKit/CleanReport.swift", "app/Packages/DDCKit/Sources/DDCKit/DiskReport.swift", "app/Packages/DDCKit/Tests/DDCKitTests/", "bench/golden/", "scripts/build-cli.sh", "Makefile", "README.md"]
specs: []
started_at: "2026-10-30T07:36:37.778Z"
closed_at: "2026-10-30T09:48:37.314Z"
plan_updated_at: "2026-10-30T07:36:37.778Z"
---

# Platform seam: ADR and capability model

| Field | Value |
|---|---|
| **Epic** | [DDC-007](../epic.md) |
| **Status** | done |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

Before any code moves, an ADR decides the shape of the seam. This story also delivers the parts that are small and independent of the move:
- the capability model in the JSON;
- the app's decoder;
- the switch that keeps an unported platform from scanning or cleaning.

What sits behind the seam (from today's code):

| Capability | macOS today | Off macOS |
|---|---|---|
| Directory listing with metadata | `getattrlistbulk` (`sys/bulk.rs`) | Linux: `getdents64` + `statx` (`rustix`); Windows: handle-based enumeration (Windows story) |
| Verified-parent file operations: open parent, stat child, rename to tombstone, `(dev, ino)` re-check, remove | `sys/fsops.rs`, `exec/delete.rs` (safety ADR §5) | Linux: the same `*at` calls; Windows: handle-based rename and dispose, `FileIdInfo` |
| Allocated size, file identity, one filesystem | `ATTR_FILE_ALLOCSIZE`, dev + inode | `st_blocks`, dev + inode / `AllocationSize`, volume serial + 128-bit file id |
| Private size | APFS clone ids (`disk/apfs.rs`) | capability absent (btrfs/ReFS reflinks out of scope) |
| Free space, volumes | `statfs` | `statvfs` + `/proc/self/mountinfo` / `GetDiskFreeSpaceExW` |
| Processes, open files, cwd, executables | `libproc` (`sys/procs.rs`) | `/proc/*` / Restart Manager or absent |
| Command runner: process group, timeout kill | `exec/system.rs` (`SIGKILL` to the group) | Linux the same; Windows Job Object |
| I/O policy, rlimits, signals, terminal | `sys/iopolicy.rs`, `rlimit.rs`, `signals.rs`, `term.rs` | per OS or no-op, stated |
| Home, data and log folders | `HOME`, `~/Library/…` (`env.rs`) | `HOME`/XDG; `USERPROFILE`, `%LOCALAPPDATA%` |
| Whole-disk view, Time Machine, simulators | `disk/*` | capability absent: `ddc disk` refuses with the capability named |
| Schedule, notifications | launchd, `osascript` (`ddc-cli`) | capability absent (epic Open Question) |

## Decomposition

- [x] ADR "Platform seam: one platform module per OS with declared capabilities". Read `adr/README.md` first. It decides:
  - `cfg`-selected modules with one shared signature set versus traits (proposal: `cfg` modules, no dynamic dispatch on the hot path; traits only where tests need fakes, as `exec::system` already has);
  - the capability list and its names;
  - the JSON change;
  - the `SUPPORTED` switch;
  - where `ddc-cli`'s OS code lives (`cli::platform`).
  
  Run `projectstore:critic` on it.
- [x] `platform::SUPPORTED: bool`, true only on macOS until a catalog story flips it. It is checked in the core entry points of scan, clean, du and disk, so `background` and any future caller hit it, and again at CLI dispatch. On an unported platform only `--version`, `--help` and `rules` run. Every other command (`scan`, `clean`, `du`, `disk`, `background`, `schedule`, `notify`, `diag`) exits with a dedicated exit code.
- [x] One JSON error envelope for every `--json` refusal: today the CLI has none. The ADR defines it (for example `{ "error": { "code": "platform-not-supported", "message": "…", "platform": "<os>" } }`) with `docs/error.schema.json`, and the platform refusal is its first user. Source builds (`cargo install`, binstall's compile fallback) hit the same switch, so a half-ported platform cannot delete anything.
- [x] JSON: `platform` and `capabilities[]` in the scan, clean and disk reports; a `schema_version` bump; `docs/*.schema.json` updated.
- [x] The app's decoder (DDCKit) accepts the new fields; contract tests updated (`swift test`).

## Implementation Plan

The how is ADR [[adr/platform-seam-one-platform-module-per-os-with-declared-capabilities|Platform seam: one platform module per OS with declared capabilities]] (accepted 2026-10-30). This is the route through it.

1. `ddc-core/src/platform/` (ADR §1–§3, §5): `mod.rs` with `Platform`, `Capability` (ten, kebab-case), `PLATFORM`, `CAPABILITIES`, `SUPPORTED` (the `cfg!` form), `Unsupported { platform, simulated }`, `require_supported()`, `refusal_in()`, the `cfg` selection and `compile_error!`; `macos.rs` (`true`, all ten), `linux.rs` and `windows.rs` (`false`, `&[]`). The feature `simulate-unported` in `ddc-core`, forwarded by `ddc-cli`.
2. Core gates (ADR §5): `walk_with_reader`, `scan_inner`, `disk::measure_with` (now a `Result`), `exec::preflight::preflight`, `exec::execute`, `free::run`; the `Unsupported(Unsupported)` variants of `WalkSetupError`, `ScanError` and `ExecError`; `io::Error::new(ErrorKind::Unsupported, …)` for `preflight` and `free::run`.
3. JSON (ADR §4): `platform` and `capabilities` after `ddc_version` in `ScanReport`, `CleanReport` and `DiskReport`; versions 1→2, 3→4, 2→3; the `full_disk_access` doc; `docs/*.schema.json` regenerated; `make golden-record`, with a check that the golden diff adds only the two fields and the version.
4. The envelope (ADR §7): `ddc_core::refusal::ErrorReport` with `error_schema_version`, `docs/error.schema.json` and its drift test.
5. CLI (ADR §5–§6): `exit::UNSUPPORTED = 69`; the dispatch gate after clap and before `init_process`, an exhaustive match over `Command`, `rules` the only command allowed; the envelope on stdout under the JSON flags; `refusal_in` in `main`.
6. Tests (ADR §8–§9): `make unported-check` (into `target/unported`) in `make gate`, with `ddc-core/tests/unported.rs` and `ddc-cli/tests/unported_cli.rs`; structure tests for the feature, the environment reads and the literal `SUPPORTED`/`CAPABILITIES`; a macOS test that `SUPPORTED` is true; the skip helper in `tests/common`; the scan smoke in `scripts/build-cli.sh`.
7. DDCKit (ADR §4): `Platform` and `Capability` decoded strictly in the scan, clean and disk models; the fixtures and the contract test updated; `swift test`.
8. Docs: README's exit-code table gains 69; the menu bar ADR §6 item 5 gains 69 (amendment); the move story's acceptance criterion names `ddc-cli/src/platform/`.

## Acceptance Criteria

- [x] The ADR is accepted after critic review. — evidence: ADR "Platform seam: one platform module per OS with declared capabilities", `status: accepted`, `review_status: approved`, 2026-10-30
- [x] On macOS, reports carry `platform: "macos"` and the full capability list; every other field is unchanged (golden check from DDC-005 story "Synthetic load trees: large, wide, deep and hostile"). — evidence: `bench/golden/` diff in 9ce7b80 adds only `"platform": "macos"`, the ten capabilities and the version (scan 1→2, disk 2→3); `crates/ddc-core/tests/platform.rs` `macos_is_supported_with_every_capability`; golden check inside the pre-push `make gate` (613 tests passed) and Gate run 41035955267
- [x] A test build with `SUPPORTED = false` refuses every command except `--version`, `--help` and `rules` with the documented exit code, and with JSON that validates against `docs/error.schema.json` where `--json` was given. A library call to the core's scan entry point is refused too. — evidence: `make unported-check` in the pre-push gate of 9ce7b80, 10 tests passed: `crates/ddc-cli/tests/unported_cli.rs` `every_command_but_rules_is_refused_with_69` (every command and JSON mode, envelope validated against `docs/error.schema.json`), `version_help_and_rules_run`, `a_usage_error_is_still_2`; `crates/ddc-core/tests/unported.rs` `a_scan_is_refused_before_it_reads_the_home` and the walk, disk, execute, preflight and `free::run` cases
- [x] `swift test` passes with the new schema version. — evidence: `swift test` in `app/Packages/DDCKit` on the tree of 9ce7b80, 379 tests in 41 suites passed (`ContractTests` platform contracts, the schema-version tests); the app job of Gate run 41035955267

## Final Summary

**What changed.** `ddc-core` gained `platform/`: `PLATFORM`, `SUPPORTED` and ten declared capabilities per OS, with macOS supported and declaring all ten, and Linux and Windows not yet supported. Every core entry point checks `SUPPORTED` first: scan, walk, disk, execute, preflight and `free::run`. The CLI refuses at dispatch with exit 69 (`EX_UNAVAILABLE`); only `--version`, `--help` and `rules` run on an unsupported platform. Under the JSON flags it prints the error envelope of `docs/error.schema.json`. The scan, clean and disk reports carry `platform` and `capabilities` (schemas 2, 4 and 3), and DDCKit decodes them strictly. `make unported-check` builds with the `simulate-unported` feature and joins `make gate`; `scripts/build-cli.sh` smoke-scans the release binary so a simulated build cannot ship. Commit 9ce7b80.

**Why.** DDC-007 ports the CLI to Linux and Windows. The seam fixes the shape before any code moves, and the switch keeps a half-ported platform, source builds included, from scanning or cleaning.

**Tests executed.** Pre-push `make gate` on 9ce7b80: 613 tests passed, plus `unported-check` (10 passed) and the install tests. `swift test`: 379 passed. Gate run 41035955267 on 9ce7b80. Reviewer: all four criteria met, no blocker. Critic accepted the ADR and, after three passes, the vault edits.

**Departures from the ADR, accepted by the reviewer.** `free::dry_run` returns `Result<…, CapabilityAbsent>`. `FreeContext` carries `capabilities`, which tests inject. The dispatch gate lives in `ddc-cli/src/gate.rs`. `simulated` is `cfg!(feature)`.

**Risks and follow-ups.**
- The §9 skip helper (`tests/common::unsupported`) has no callers yet. Story "Move OS calls behind the platform module" applies it, and stage 2 of story "Build matrix: every target built, tested and packaged from one tag" waits for that.
- The §3 "when absent" behaviours are unimplemented, because no supported platform lacks a capability. They are items in the Linux and Windows catalog stories, together with the safety ADR amendment Windows needs if it lacks `in-use-check`.
- `FreeContext.capabilities` is `pub`, so a library caller could get around the per-action check (but not `SUPPORTED`). Revisit when a supported platform lacks `whole-disk` or `homebrew`.
- The menu bar ADR's status is still `proposed`, after many amendments. That is a question for the owner.

## Technical Notes

- Keep the switch a constant per platform module, not a runtime flag or an environment variable. Nothing a user or an agent sets can turn it on.

## Dependencies

- DDC-005 story "Mandatory gate before every commit".

## Attachments

-

---

*Last updated: 2026-10-30*

---
type: epic
id: "DDC-001"
title: "Disk space cleaner CLI for developers"
status: in-progress
priority: p1
created: 2026-10-17
updated: 2026-10-17
external_refs: {}
tags: [cli, rust, cleanup]
code_refs: ["Cargo.toml", "Makefile", "rust-toolchain.toml", "clippy.toml", "rustfmt.toml", "crates/ddc-core/", "crates/ddc-cli/", "docs/scan.schema.json", "bench/"]
review_status: reviewed
reviewed_at: 2026-10-17
---

# DDC-001: Disk space cleaner CLI for developers

| Field | Value |
|---|---|
| **Status** | in-progress |
| **Priority** | p1 |
| **Created** | 2026-10-17 |
| **Updated** | 2026-10-17 |

---

## Goal

In one fast pass, the `ddc` utility finds all dev junk in the home directory (project environments and dependencies, package manager caches, VMs and containers, toolchains, models, IDE data, application caches), groups it by risk class, category and project, counts the freeable space honestly and, after confirmation, carefully deletes what was selected. The scheduled background mode only scans and notifies.

Success criteria (measured from Terminal on the reference machine; median of three runs after one warm-up run):

- `ddc scan ~` finishes in ≤ 60 s, `ddc scan ~/Projects` in ≤ 20 s;
- every path from `bench/baseline-2026-10-17.tsv` that still exists is found with the expected rule (the path equals the baseline path or is nested in it, or the other way round), and the total size found is ≥ 90% of the baseline, recomputed with `du -sk` on the day of the check;
- `ddc clean` frees what was selected without touching a single protected folder, a single `user-data` item or a single VM directory.

The report states its completeness honestly: the `full_disk_access` field and the `inaccessible[]` list. Without Full Disk Access part of `~/Library` is inaccessible, and that is not an error.

## Context

The reference machine has 84 GB free. `.venv` and `node_modules` alone take 38 GB, caches about 30 GB, five container runtimes about 57 GB. Existing tools each cover a single ecosystem or do not understand what they are looking at. The data and conclusions are in [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]].

Decisions the epic relies on:

- [[adr/rust-core-library-and-cli-with-json-output|Rust core: library + CLI with JSON output]]
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]

The epic is the foundation of the future Mac app: `ddc-core` and the JSON schema will become its core and its contract.

## Stories

| Story | Status | Description |
|-------|--------|-------------|
| [story-workspace-skeleton-and-parallel-fs-scanner](./stories/story-workspace-skeleton-and-parallel-fs-scanner.md) | done | Workspace skeleton and parallel filesystem scanner: allocated blocks, hard-link deduplication, no symlinks, one filesystem |
| [story-detection-rule-catalog](./stories/story-detection-rule-catalog.md) | done | Rule catalog: project artifacts, global caches, VMs, toolchains, models, IDEs, application caches, user data |
| [story-ddc-scan-report](./stories/story-ddc-scan-report.md) | done | `ddc scan` report: grouping, project activity, large files, JSON with `schema_version` |
| [story-safe-cleanup-ddc-clean](./stories/story-safe-cleanup-ddc-clean.md) | done | Safe cleanup `ddc clean`: plan, selection, native commands, Trash, re-check, log |
| [story-scheduled-background-scan](./stories/story-scheduled-background-scan.md) | done | Background scan: LaunchAgent, saved report, threshold notification |
| [story-duplicates-and-apfs-private-size](./stories/story-duplicates-and-apfs-private-size.md) | planned | Duplicates and the exact size of a clone union: duplicate search (photos, downloads) and counting by `ATTR_CMNEXT_CLONEID` when several findings are selected |
| [story-homebrew-tap-and-the-v010-release](./stories/story-homebrew-tap-and-the-v010-release.md) | in-progress | Homebrew tap and the v0.1.0 release: signed, notarized cask in `smartandpoint/tap`, `make dist` and publish script, stable Homebrew link for the LaunchAgent, reference machine moved to the Homebrew install |
| [story-progress-output-while-ddc-clean-runs](./stories/story-progress-output-while-ddc-clean-runs.md) | in-progress | Progress output while `ddc clean` runs: the preparing phase (keep-marker walk) and items processed, bytes processed and the current item; an idea from the first real clean |

## Expected Results

- [ ] `ddc scan` — a grouped report of dev junk: "allocated" for each finding; "frees at least" (`private_bytes`) where APFS clones are expected (uv, pnpm, bun); the "frees" total is an estimate (`~`). The binary is `deepdrivecleaner`; the `ddc` command is a symlink to it
- [ ] `ddc scan --json` — a stable schema (`docs/scan.schema.json`) for the background scan and the future GUI
- [ ] `ddc clean` — interactive cleanup by plan, with protection rules and a log
- [ ] `ddc schedule install` — a periodic background scan with a notification
- [ ] Unit and integration tests on temporary directory trees; no test touches the real `$HOME`

## Dependencies

- rustup stable (≥ 1.89 — `File::try_lock` for the data directory lock; edition 2024), version pinned in `rust-toolchain.toml`; call `~/.cargo/bin/cargo` (the Homebrew `cargo` on the reference machine is broken)
- macOS 13+ on APFS (`getattrlistbulk`, `ATTR_CMNEXT_PRIVATESIZE`, LaunchAgent, `osascript`)

## Open Questions

- [ ] How exactly the GUI will use the core: calling `ddc --json` or UniFFI — to be decided in a separate ADR when the app work starts
- [ ] Full Disk Access for the app: for an unsigned CLI the permission is tied to the binary's hash and is lost on every rebuild; a LaunchAgent does not inherit the terminal's FDA. To be decided when packaging the app (signing, Developer ID)
- [x] Default notification threshold: 10 GB freeable or < 10% free space; notify again if the amount has grown by ≥ 20% or ≥ 7 days have passed

## Related

- [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]]
- [[adr/rust-core-library-and-cli-with-json-output|Rust core: library + CLI with JSON output]]
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]

---

*Last updated: 2026-10-17*

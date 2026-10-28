---
type: story
id: "story-platform-seam-adr-and-capability-model"
epic: "DDC-007"
title: "Platform seam: ADR and capability model"
status: planned
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-28
external_refs: {}
tags: [rust, architecture, portability, adr, schema]
code_refs: []
specs: []
started_at: null
closed_at: null
plan_updated_at: null
---

# Platform seam: ADR and capability model

| Field | Value |
|---|---|
| **Epic** | [DDC-007](../epic.md) |
| **Status** | planned |
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

- [ ] ADR "Platform seam: one platform module per OS with declared capabilities". Read `adr/README.md` first. It decides:
  - `cfg`-selected modules with one shared signature set versus traits (proposal: `cfg` modules, no dynamic dispatch on the hot path; traits only where tests need fakes, as `exec::system` already has);
  - the capability list and its names;
  - the JSON change;
  - the `SUPPORTED` switch;
  - where `ddc-cli`'s OS code lives (`cli::platform`).
  
  Run `projectstore:critic` on it.
- [ ] `platform::SUPPORTED: bool`, true only on macOS until a catalog story flips it. It is checked in the core entry points of scan, clean, du and disk, so `background` and any future caller hit it, and again at CLI dispatch. On an unported platform only `--version`, `--help` and `rules` run. Every other command (`scan`, `clean`, `du`, `disk`, `background`, `schedule`, `notify`, `diag`) exits with a dedicated exit code.
- [ ] One JSON error envelope for every `--json` refusal: today the CLI has none. The ADR defines it (for example `{ "error": { "code": "platform-not-supported", "message": "…", "platform": "<os>" } }`) with `docs/error.schema.json`, and the platform refusal is its first user. Source builds (`cargo install`, binstall's compile fallback) hit the same switch, so a half-ported platform cannot delete anything.
- [ ] JSON: `platform` and `capabilities[]` in the scan, clean and disk reports; a `schema_version` bump; `docs/*.schema.json` updated.
- [ ] The app's decoder (DDCKit) accepts the new fields; contract tests updated (`swift test`).

## Implementation Plan

## Acceptance Criteria

- [ ] The ADR is accepted after critic review.
- [ ] On macOS, reports carry `platform: "macos"` and the full capability list; every other field is unchanged (golden check from DDC-005 story "Synthetic load trees: large, wide, deep and hostile").
- [ ] A test build with `SUPPORTED = false` refuses every command except `--version`, `--help` and `rules` with the documented exit code, and with JSON that validates against `docs/error.schema.json` where `--json` was given. A library call to the core's scan entry point is refused too.
- [ ] `swift test` passes with the new schema version.

## Final Summary

## Technical Notes

- Keep the switch a constant per platform module, not a runtime flag or an environment variable. Nothing a user or an agent sets can turn it on.

## Dependencies

- DDC-005 story "Mandatory gate before every commit".

## Attachments

-

---

*Last updated: 2026-10-28*

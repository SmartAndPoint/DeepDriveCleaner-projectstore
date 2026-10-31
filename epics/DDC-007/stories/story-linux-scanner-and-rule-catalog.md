---
type: story
id: "story-linux-scanner-and-rule-catalog"
epic: "DDC-007"
title: "Linux scanner and rule catalog"
status: in-progress
priority: p2
assignee: "Evgenii Konev"
created: 2026-10-28
updated: 2026-10-31
external_refs: {}
tags: [rust, linux, scanner, rules, safety]
code_refs: []
specs: []
started_at: "2026-10-31T05:03:08.564Z"
closed_at: null
plan_updated_at: "2026-10-31T05:03:08.564Z"
---

# Linux scanner and rule catalog

| Field | Value |
|---|---|
| **Epic** | [DDC-007](../epic.md) |
| **Status** | in-progress |
| **Priority** | p2 |
| **Assignee** | Evgenii Konev |

---

## Description

This story fills `platform::linux` and the Linux catalog section, so `ddc scan` and `ddc clean` work on a Linux developer's home with the same risk classes, confirmations and verified-parent deletes as on macOS. Then it flips Linux to supported and publishes the Linux archives.

Before the catalog is written, a research note records where dev disk space goes on Linux, with measured sizes. It plays the role [[research/where-developer-disk-space-goes-macos|Where a developer's disk space goes (macOS)]] plays for macOS.

## Decomposition

- [x] Research note [[research/where-developer-disk-space-goes-linux|Where a developer's disk space goes (Linux)]], critic-approved 2026-10-31. Measure on a real or VM Ubuntu developer setup:
  - XDG cache and data (`~/.cache`, `~/.local/share`);
  - per-tool caches (pip, uv, npm, pnpm, yarn, bun, cargo, go, gradle, maven, Hugging Face, JetBrains, VS Code);
  - rootless Docker and Podman storage under `~/.local/share` (VM class: reported, never deleted; root Docker is outside the home);
  - Flatpak and Snap user data.
  
  Run the critic on it.
- [x] Walker: `getdents64` + `statx` through `rustix`. Allocated size from `stx_blocks`. Hard links via (dev, ino). One filesystem via `stx_dev`. Symlinks never followed. — done: code PRs #31, #32; a folder's own blocks count as `du` counts them, #52, #54
- [x] Verified-parent operations (`openat`, `fstatat`, `renameat` to a tombstone, re-check, `unlinkat`) as on macOS, using the same safety ADR §5 contract. — done: #32
- [x] Free space and volumes: `statvfs` and `/proc/self/mountinfo`. — done: #34
- [x] In-use checks with the same semantics as macOS (safety ADR §5):
  - `/proc/*/fd`;
  - `/proc/*/cwd`;
  - `/proc/*/exe`;
  - `/proc/*/cmdline`;
  - `/proc/*/environ` (own processes only; `VIRTUAL_ENV`).
  
  Details a process withholds are left empty, as on macOS, and counted in the clean report (safety ADR amendment of 2026-10-31, which also states the Linux limits). — done: #34, #35; the count, #53
- [x] Trash: freedesktop Trash via the `trash` crate, home and other-volume trash. — done: #35; the real-Trash CLI test, #53. The other-volume path has no automated test: it goes to the owner checks
- [x] Catalog (Linux section): rules from the research note. Protected folders: the safety ADR §6 list (it already holds `~/.config`, `~/.ssh`, `~/.gnupg`, `~/.local`, `~/.cache` as protected folders, not as their contents), plus Linux-only folders and Linux system folders (`/`, `/home`, `/usr`, `/etc`, `/opt`, `/var`, `/boot`; the shared section from story "Rule catalog split: shared and per-OS sections" has none), amended into the safety ADR §6 before `SUPPORTED` flips (the catalog split's flip-guard test refuses the flip otherwise). The rules JSON gains its `platform` field and schema bump with this flip (seam ADR §4 amendment of 2026-10-30). Linux-only folders: `~/.local/share/keyrings`, `~/.local/share/Trash`, `~/.pki`, browser and mail profiles (`~/.mozilla`, `~/.thunderbird`). — done: protected folders #36; rules #42, #44, #48; `platform` field #46; `/var/lib/docker` never walked #53
- [x] Home, data and logs: `HOME`; `$XDG_STATE_HOME/deepdrivecleaner` (default `~/.local/state/deepdrivecleaner`), through `env::Env`. — done: #31
- [x] JSON paths: Linux allows names that are not UTF-8, and APFS refuses them. Today `scan --json` would fail outright on one in a reported path (a finding, a listed entry, an error or skipped path, a root or the home; a name strictly inside a claimed finding never reaches the JSON), because serde refuses to serialize a non-UTF-8 `PathBuf`. Decide how the schema represents such a name (lossy with a flag, or escaped bytes), implement it, and check that the `names` load profile, which builds invalid UTF-8 where the filesystem accepts it, passes on Linux. — done: #40 (lossy strings, `lossy_paths`)
- [ ] Load thresholds per target: Linux has no bulk stat (one `statx` per entry) and musl's allocator is slow under threads. Measure, then set Linux thresholds; try `mimalloc` for musl if the S check misses by more than 2×.
- [x] Honour the "when absent" column of ADR "Platform seam: one platform module per OS with declared capabilities", §3, for every capability this platform does not declare: `notifications` (`scan --background` decides but does not notify), `full-disk-access` (`full_disk_access` is true), `private-size` (no `private_bytes`) and `local-snapshots` (`local_snapshots` is 0). Story "Platform seam: ADR and capability model" left them unimplemented: no supported platform lacked one. `dataless-files` is absent on Linux: the catalog lists the cloud-sync roots it does not walk (Dropbox, Nextcloud, Insync and the like), or the research note states that none needs listing. — done: #35; the research note §8 states none needs listing
- [x] Flip `SUPPORTED` to true for Linux and add the Linux targets to `release.yml`'s enabled list, in one commit. — done: #55 (0.13.11)

## Implementation Plan

Planned 2026-10-31 with the planner, from origin/main 243733e. Each step is its own pull request, green on all seven required checks, iterated on Linux with `scripts/native-linux.sh` before CI.

**Gate before code (vault):** the safety ADR's, seam ADR's and Rust core ADR's 2026-10-31 amendments (Linux §5/§6, `platform/linux/` with `unsafe`, Linux declares `in-use-check` only, non-UTF-8 paths as lossy strings plus `lossy_paths`), critic-approved with this plan.

**Why most skips leave only at the flip.** About 400 of `.config/native-skips/linux.txt`'s names skip through `common::unsupported()` or `platform::skipped()`, which key on `SUPPORTED`, and the seam ADR forbids any switch that turns it on early. Before the flip a step shrinks the list in two ways only:
- re-gating a test on the primitive it needs (`skipped_without(test, <probe>)`, still `Unsupported` on Windows, so `windows.txt` keeps the name);
- new unit tests under `platform/linux/`, which run at once.

Each step also proves itself with a dry flip: `SUPPORTED = true` edited locally, never committed, and the affected test files run on Linux. The pull request names the count.

1. **Foundation.** `platform/linux.rs` becomes `platform/linux/mod.rs`, with these submodules on `rustix` (no `unsafe`): `meta`, `errno`, `testing` (plus a `posix_paths()` probe, unsupported in `unported.rs`), `folders` (`HOME`; `$XDG_STATE_HOME/deepdrivecleaner`), `rlimit`, `term`, `process`.
   - `rustix` becomes a Linux dependency of `ddc-core`.
   - The path-shape tests the technical notes list are re-gated on `posix_paths()`.
   - `structure.rs` and `catalog/mod.rs` now read `linux/mod.rs`.
2. **plutil fixture bytes.** The two binary plists are committed. `report::backups` reads them with `include_bytes!`, and `testing::plutil_to_binary` goes.
3. **Walker** (`platform/linux/bulk.rs`):
   - `openat`, then `statx` on the directory (identity and the `expect` check), then `getdents64`, then `statx` per entry with `AT_SYMLINK_NOFOLLOW | AT_STATX_DONT_SYNC`;
   - `dev` from `makedev`, `alloc = stx_blocks × 512` (directories 0), `nlink`, and the owner `uid` that step 10 reads;
   - `fstatat` where `statx` is refused.
   
   Tests compare it with `StdReader` on a temp tree: a symlink is not followed, a hard link, a non-UTF-8 name, an unreadable directory, a swapped directory.
4. **Verified-parent operations** (`platform/linux/fsops.rs`): `DirFd` on `openat`, `stat_child` by `statx`, `rename_child` by `renameat2(NOREPLACE)` with a fallback, `unlink_child`, `set_times` (mtime only), `can_write`. A `remove()`-level test of a directory swapped for a symlink.
5. **Volumes** (`platform/linux/statfs.rs`): `statvfs`, and `/proc/self/mountinfo` parsed with its octal escapes. Fixture lines cover a bind mount, overlay and a btrfs subvolume.
6. **Usage and signals.** `unsafe` is allowed in `platform/linux/`, and the structure test's allowlist widens with it.
   - `wait4` and `getrusage` (RSS in KiB);
   - thread CPU time;
   - `/proc/loadavg`;
   - the DMI product name;
   - signals.
7. **In-use** (`platform/linux/procs.rs`):
   - the current user's processes from `/proc`: `exe`, `cmdline`, `cwd`, `environ`, `fd/*`, `maps`;
   - the name from the executable or `argv[0]`, not `comm`;
   - `real_home` from `/etc/passwd`, falling back to `HOME` for users only NSS knows (LDAP, sssd);
   - the start time from `stat` and `btime`.
   
   Tests: a child's working folder, a held-open file, a venv's `bin/python` found through `cmdline`.
8. **Trash** (`platform/linux/trash.rs`): the `trash` crate is now a dependency on macOS and Linux. Tests cover the home trash and another filesystem's `.Trash-$uid`, and what the crate does when neither trash works.
9. **The "when absent" column.** Shared code, tested with an injected capability list:
   - `full_disk_access` is true without the capability;
   - `local_snapshots` is 0;
   - no `want_private`;
   - no notifier call.
   
   The tests of the disk, brew, agents, schedule and notifications commands move from `unsupported()` to `lacks(Capability::…)`; the skip names stay.
10. **Ownership refusal** (the safety ADR amendment):
    - `EntryMeta.uid`, filled by the step 3 walker;
    - the scan lists a finding with a foreign owner and never selects it;
    - the guard's re-walk refuses it before any rename;
    - `make_dirs_writable` skips folders the user does not own;
    - the new reason `foreign-owner` in `not_selected` and `skipped[].reason` moves the scan and clean schema versions, with the docs schemas and DDCKit.
    
    Tests use a fake reader with a foreign uid, so they run on every OS.
11. **Non-UTF-8 JSON paths** (the Rust core ADR amendment):
    - `json_path.rs` with a `serialize_with` on every output `PathBuf`, and `lossy_paths` in the scan and clean reports;
    - `settings.json` refuses a non-UTF-8 path at write and keeps its own serializer;
    - a guard test that no output path lacks it;
    - the `names` load profile run with a dry flip.
12. **Linux protected folders** in `rules/catalog/linux.rs`, as the safety ADR amendment lists them; `catalog/mod.rs`'s `!may_be_supported(Platform::Linux)` assertion turns.
13. **Catalog rules:**
    - (a) narrow `model.huggingface` to `hub`, `xet` and `datasets` on both platforms;
    - (b) remote editor servers, `Removal::Delete` so the in-use check counts open files: `.staging*` leftovers, `bin/<commit>` builds and the `cli/servers` builds past the head of `lru.json`; the keep sits beside `NEWEST_KEPT` in `rules/toolchain.rs`, and live builds are left to the in-use check;
    - (c) Linux JetBrains rules: caches, old-version data, Gateway's `RemoteDev/dist`;
    - (d) VM-class rules for rootless Docker (`~/.local/share/docker`) and Docker Desktop for Linux (`~/.docker/desktop`).
    
    The catalog test that holds Linux equal to Windows and in macOS order is relaxed.
14. **Load thresholds per OS:** `bench/load-limits-linux.tsv`, read by `ddc-loadgen`'s `limits.rs` on Linux and measured on Linux. `mimalloc` for musl only if the S check misses by more than 2×.
15. **The rules JSON `platform` field** and its schema bump (seam ADR §4 amendment), as its own commit. Until it lands, `ddc rules --platform linux` prints the Linux-only rules of step 13 under schema 1, which is acceptable while Linux is unsupported.
16. **The flip, in one commit:**
    - `SUPPORTED = true` and `CAPABILITIES = [in-use-check]` for Linux;
    - `x86_64-unknown-linux-musl` and `aarch64-unknown-linux-musl` in `release.yml`'s `RELEASE_TARGETS`;
    - `linux.txt` burned down to the `lacks()` names.
    
    Before the push, the musl build and its smoke test run on Linux, since CI tests only the gnu target.

**Decided at this gate:**
- The XDG cache-root token (research note, Conclusion 1) is not part of the flip; it gets a follow-up story. No override was measured, and a missed override only under-reports.
- Root Docker's own numbers are not reported on Linux: there is no `whole-disk` there. Follow-up story.
- Deferred, per the research note: Flatpak and Snap, and Claude Code's `versions` folder (coding agents' data).
- The AC's "manual VM run" goes to the owner-checks runbook.

**Proposed code_refs:** `crates/ddc-core/src/platform/linux/`, `crates/ddc-core/src/platform/`, `crates/ddc-core/src/rules/catalog/`, `crates/ddc-core/src/rules/toolchain.rs`, `crates/ddc-core/src/scan/`, `crates/ddc-core/src/exec/`, `crates/ddc-core/src/json_path.rs`, `crates/ddc-loadgen/src/limits.rs`, `bench/load-limits-linux.tsv`, `.config/native-skips/`, `.github/workflows/release.yml`, `docs/`, `app/Packages/DDCKit/Sources/DDCKit/`.

### Progress, 2026-10-31: Linux supported

Steps 1-13 and 15-16 merged as code PRs #27-#55; Linux is supported from 0.13.11 (#55). Each step was iterated on a Linux machine with `scripts/native-linux.sh` and dry flips. On the flip, Linux's skip list went from 441 names to 113, each skipped by design: capabilities Linux does not declare, rules only macOS lists, macOS facts, and the goldens and pinned trees recorded on macOS. The Linux leg (712 tests, 113 skipped by design) and GitHub's Ubuntu runner were green; macOS 702/702. `release.yml`, dispatched on main at 0.13.11, built, smoke-tested and attested both Linux archives on their own runners (x86_64 on ubuntu-24.04, aarch64 on ubuntu-24.04-arm).

What the work did beyond, or other than, the plan:
- **Shared rules with per-OS paths, not Linux-own JetBrains rules.** The JetBrains, VS Code, Cursor and Windsurf caches, workspace storage and Docker Desktop became shared rules with per-OS paths (#44). So did the coding agents' rules, the Chrome, Brave, Firefox, Spotify and Slack caches and the Trash (#48). A rule keeps one id on every OS, and the sibling keep of `ide.jetbrains-old-data` works unchanged. This reverses story "Rule catalog split: shared and per-OS sections"'s "every rule with `owners` is macOS-only": owners stay macOS bundle names, which match no process elsewhere, and the open-file check of a `Delete` stands in for them. `Matcher::KnownChildren` takes several parents, and the sibling keeps decide in each parent alone.
- **Remote editor servers** (`ide.vscode-server`, `ide.cursor-server`) are shared rules, since VS Code reaches macOS and Windows hosts too. The keep is the head of `lru.json` in `cli/servers`, and the newest build in `bin`.
- **A folder's own blocks count** (#52, #54). ext4 gives a folder 4 KiB and more, and a long symlink a block; `du` counts both. The walk counts a folder once: from its own listing, or from its parent's entry when it cannot be read. Without this, the synthetic `projects` load tree read 40% low. Read-only checks with a dry-flipped static build on two Linux homes (x86_64 and aarch64) matched `du -skx` to the KiB on every folder compared and on the whole home, and the walk was faster than `du`. The machines are private: their numbers stay with the owner.
- **The date added** is the `statx` birth time on Linux, where the filesystem keeps one (#47).
- **Tests** build their fixtures where each platform's catalog looks (`rule_path`), and skip a macOS-only rule through `no_rule` (#51). The load trees are built from the platform's own catalog, with Linux's 4096-byte `PATH_MAX`. The verifier reads folder and symlink blocks from the built tree, apart from the scan (#52). The seam ADR's amendment of 2026-10-31 to §9 names the new channels.
- **The `trash` crate** tries `$topdir/.Trash/$uid`, then creates `$topdir/.Trash-$uid`. Only when the volume's root is not writable does it copy the item into the home trash: the source goes, the home filesystem gains the item's size, and a copy that fails halfway leaves a partial copy in the Trash with the source intact. The safety ADR's §5 records this (amendment of 2026-10-31, completed at the flip). No automated test covers a second volume; the owner checks do.
- **The coding agents' rules are live on Linux**, while the report-level liveness tests (`agents.rs`) skip there through `lacks(coding-agents)`, which covers `ddc disk agents` only. What guards a clean is not capability-gated: the plan's `agent-live` check runs on Linux, and `agents/running.rs` matches the harnesses by executable name. The `liveness.rs` unit tests, gated on `posix_paths()`, run on Linux and prove it.

Left in this story: step 14 (Linux load thresholds); the pre-release that publishes both Linux archives (stage 3 of story "Build matrix: every target built, tested and packaged from one tag"); the manual VM run, for the owner checks.

## Acceptance Criteria

- [ ] The research note exists, critic-approved, with measured sizes.
- [ ] On Ubuntu (CI and a manual VM run), `ddc scan ~ --json` on a seeded home finds the catalog's caches and project artifacts and validates against the schema.
- [ ] DDC-005's property tests pass on Linux: no protected folder, `user-data` item or VM directory is ever planned; symlinks are never followed.
- [ ] A folder swapped for a symlink between scan and clean is skipped as changed since the scan, not followed (test).
- [ ] `ddc clean` moves a selected cache to the freedesktop Trash and the journal records it (integration test in CI).
- [ ] A pre-release tag publishes both Linux archives, and `install.sh` installs them (story "One-line install on macOS and Linux: ddc.sh/install.sh and cargo").

## Final Summary

## Technical Notes

- *(Superseded by the Progress note where they disagree: the skip list after the flip and its count.)*
- `ext4` and `btrfs` report allocated size the same way. btrfs reflinks are like APFS clones: private size is out of scope, stated as an absent capability.
- Iterate on a Linux machine with docker through `scripts/native-linux.sh`, not through CI rounds. It runs the Gate's Linux leg in about a minute: clippy, nextest with profile `ci`, doc tests and `scripts/native-skips.sh linux`, in `rust:<toolchain>-bookworm` as a non-root user. Root reads unreadable folders, so it changes results.
- The flip burns down `.config/native-skips/linux.txt`. It held 431 names when the build matrix's stage 2 merged, and it grows with each new test that skips off macOS. After it, only skips through `lacks()` should remain: Homebrew, the whole disk, the coding agents. Plus whatever the research note shows Linux does not have.
- Stage 2 also skipped some tests on Linux that pass there:
  - `there_is_no_yes`;
  - the macOS-path tests (`agents::running`, `agents::liveness`, `free::probe`, `keep`, the malformed-list test);
  - the fuzz regressions' "accepted" half.

  They come back with the flip, unless they are re-gated on path shape first. The in-crate ones (`keep`, `agents::running`, `agents::liveness`, `free::probe`) skip through `platform::skipped()`. `there_is_no_yes`, the malformed-list test and the fuzz half skip through `common::unsupported()`.
- `report::backups`' truncation and bit-flip tests get their binary property lists from `/usr/bin/plutil`, so they skip without it. To keep that coverage after the flip, commit the two binary plists as fixture bytes.

## Dependencies

- Stories "Move OS calls behind the platform module" and "Rule catalog split: shared and per-OS sections".
- Story "Build matrix: every target built, tested and packaged from one tag" (stage 2, and `release.yml` with its `enabled` list from stage 1).

## Attachments

-

---

*Last updated: 2026-10-31*

---
type: epic
id: "DDC-010"
title: "Linux at parity with macOS: every command, every file kind"
status: in-progress
priority: p1
created: 2026-11-01
updated: 2026-11-01
external_refs: {}
tags: [linux, cli, platform, parity, scanner, safety]
code_refs: ["scripts/ci-mounts.sh", "scripts/native-skips.sh", "scripts/gate-scope.sh", ".github/workflows/gate.yml", ".github/workflows/nightly.yml", ".config/native-skips/linux.txt", "crates/ddc-core/tests/common/mounts.rs", "crates/ddc-core/tests/common/mod.rs", "crates/ddc-core/tests/mounts.rs", "scripts/tests/test_native_skips.py", "scripts/tests/test_gate_scope.py"]
review_status: approved
reviewed_at: 2026-11-01
---

# DDC-010: Linux at parity with macOS: every command, every file kind

| Field | Value |
|---|---|
| **Status** | in-progress |
| **Priority** | p1 |
| **Created** | 2026-11-01 |
| **Updated** | 2026-11-01 |

---

## Goal

Every `ddc` command that works on macOS also works on Linux. It works on every filesystem and file kind a Linux developer's disk holds, under the same safety model. "Works" means the command runs, its numbers are right, and a clean on Linux is as safe as on macOS. Refusing with exit 69 does not count as working.

Success criteria:

- **Commands.** `ddc disk`, `ddc disk free`, `ddc disk agents`, `ddc disk brew-packages`, `ddc schedule …` and the background scan's notification work on Linux. `CAPABILITIES` for Linux holds `private-size`, `whole-disk`, `coding-agents`, `homebrew`, `schedule` and `notifications`, besides today's `in-use-check`.
- **Filesystems.** The walk and the clean are correct on ext4, btrfs (subvolumes, read-only and writable snapshots, reflinks), XFS, ZFS (datasets, snapshots), tmpfs, NFS and CIFS, case-insensitive filesystems (casefold ext4, exFAT) and encrypted homes. They are also correct across bind mounts and automount points.
- **File kinds.** The walk and the clean are correct for FIFOs, sockets, device nodes, immutable and append-only files, sparse files, and names a path cannot hold. Each case has a test, or is on the owner-checks runbook with the reason it cannot be automated.
- **Catalog.** The Linux catalog covers what Linux has and macOS does not: Flatpak, Snap, Steam and Linuxbrew. It finds caches a user moved with `XDG_CACHE_HOME` or a tool's own variable, and keeps the moved folders protected.
- **Skips.** `.config/native-skips/linux.txt` holds only three groups: subjects Linux lacks, capabilities not yet declared, and tests that need the CI mounts. Each group says so. The goldens and pinned trees have Linux versions.

## Context

Story "Linux scanner and rule catalog" (DDC-007) made Linux a supported platform in 0.13.11. These commands work there today:
- `scan` and `clean`;
- `du`, `rules` and `keep`;
- `notify ack/snooze`;
- `diag`.

Their sizes match `du -skx` to the KiB on two real homes. The walk already counts hard links honestly: an inode linked outside a finding goes to `shared_bytes`, not `bytes` (`scan/inodes.rs`, `scan/walker.rs`). The full test suite runs on Ubuntu CI and on the owner's x86_64 and aarch64 Linux machines, and the nightly load job has Linux limits. Everything else refuses with exit 69 (`CapabilityAbsent`), as ADR "Platform seam: one platform module per OS with declared capabilities" requires.

Linux declares `in-use-check` only (`crates/ddc-core/src/platform/linux/mod.rs`). The command gate (`crates/ddc-cli/src/gate.rs`) maps the rest:

| Capability | Gates | What macOS uses for it |
|---|---|---|
| `whole-disk` | `ddc disk`, `disk free`'s conda and simulator actions | `disk/areas.rs`: a table rooted at `/System/Volumes/Data`, plus APFS and `diskutil` |
| `coding-agents` | `ddc disk agents` | `agents/`: harness folders, including `Library/Application Support/Claude/claude-code` |
| `homebrew` | `ddc disk brew-packages`, `disk free`'s brew actions | `brew` under `/opt/homebrew` |
| `schedule` | `ddc schedule install/uninstall/status` | `crates/ddc-cli/src/platform/launchd.rs`, a LaunchAgent |
| `notifications` | the background scan's notification (`cmd/background.rs`) | `osascript` |
| `private-size` | `private_bytes` in the scan (`scan/mod.rs`) | APFS private size of cloned files |

Three capabilities stay absent on purpose:
- `full-disk-access`: Linux has no TCC, so `full_disk_access` is true.
- `dataless-files`: the Linux clients that keep placeholder files (onedriver, rclone mount, Nextcloud's virtual files) are FUSE mounts or small real files. The one-filesystem walk never enters a FUSE mount, so no walk downloads anything (research note §8).
- `local-snapshots`: APFS local snapshots. Whether btrfs and ZFS snapshots can stand in is decided in story "Whole disk on Linux: the areas outside the home".

Code reading and the critic found gaps that are not capability questions:
- The walk calls `statx` without `AT_NO_AUTOMOUNT` (`platform/linux/bulk.rs`), so it can trigger an automount.
- The one-filesystem rule compares `stx_dev` only. A bind mount of the same filesystem passes that check, so the walk enters it. A btrfs subvolume and a ZFS dataset fail it, so the walk stops at them. The walk cannot tell a snapshot from a subvolume.
- btrfs gives subvolumes anonymous device numbers that change across mounts, while `ddc` saves `(dev, ino)` pairs in `last-scan.json` and in kept-folder records.
- The `trash` crate copies files into the home Trash when a move crosses filesystems. The safety ADR accepted that on 2026-10-31. This epic makes cross-filesystem cleans common.
- Rootless containers keep image files under subordinate uids, which the user cannot read. So `vm.podman` and `vm.docker-rootless` under-count, with no signal.
- The catalog finds a cache only at its default path. Today's probes (`go env GOMODCACHE`, `uv cache dir`) check that a native command will clean the right folder. They do not find a moved one.
- 16 rules are macOS-only (`ddc rules --platform macos` against `--platform linux`, 2026-11-01). Some have Linux counterparts: `cache.homebrew`, `vm.rancher-desktop`, `cache.colima` and `user.telegram`.

## Stories

| Story | Status | Description |
|-------|--------|-------------|
| [story-linux-ci-mounts-a-filesystem-matrix-job-with-root-only-steps](./stories/story-linux-ci-mounts-a-filesystem-matrix-job-with-root-only-steps.md) | done | A CI job with loopback btrfs, XFS, ZFS, NFS, casefold ext4, exFAT, tmpfs and a bind mount, plus root-only fixtures (`chattr +i`, `mknod`); spike first |
| [story-linux-mount-boundaries-automounts-bind-mounts-and-the-trash-across-filesystems](./stories/story-linux-mount-boundaries-automounts-bind-mounts-and-the-trash-across-filesystems.md) | planned | `openat2(RESOLVE_NO_XDEV)`, `AT_NO_AUTOMOUNT`, never cross a bind mount; the Trash never copies, it uses the volume's `.Trash-$uid` or refuses |
| [story-btrfs-and-zfs-on-linux-subvolumes-datasets-snapshots-and-stable-identities](./stories/story-btrfs-and-zfs-on-linux-subvolumes-datasets-snapshots-and-stable-identities.md) | planned | Subvolumes and datasets counted, snapshots (by `parent_uuid`, read-only or not) never; saved identities stable across remounts; compression measured |
| [story-linux-file-kinds-special-files-immutable-flags-sparse-files-and-odd-names](./stories/story-linux-file-kinds-special-files-immutable-flags-sparse-files-and-odd-names.md) | planned | FIFOs, sockets and devices never opened; `chattr +i/+a` refused before the rename; sparse files; names past `PATH_MAX`; case and Unicode as each filesystem compares them |
| [story-network-and-encrypted-homes-on-linux-nfs-cifs-fscrypt-and-ecryptfs](./stories/story-network-and-encrypted-homes-on-linux-nfs-cifs-fscrypt-and-ecryptfs.md) | planned | Squashed ids reported once, `.nfs*` silly renames, latency measured; eCryptfs lower folder never walked |
| [story-coding-agents-on-linux-ddc-disk-agents](./stories/story-coding-agents-on-linux-ddc-disk-agents.md) | planned | Harness folders at their Linux paths, Claude Code's `versions` on both OSes; declares `coding-agents` |
| [story-scheduled-scans-on-linux-through-a-systemd-user-timer](./stories/story-scheduled-scans-on-linux-through-a-systemd-user-timer.md) | planned | Units linked into the user manager by absolute path, `Persistent=true`, a refusal with a cron line where no user systemd runs; declares `schedule` |
| [story-desktop-notifications-on-linux-through-the-freedesktop-notification-service](./stories/story-desktop-notifications-on-linux-through-the-freedesktop-notification-service.md) | planned | D-Bus `org.freedesktop.Notifications` from the static binary; logged only where no server answers; declares `notifications` |
| [story-cache-roots-moved-by-the-environment-xdg-and-each-tools-own-variable](./stories/story-cache-roots-moved-by-the-environment-xdg-and-each-tools-own-variable.md) | planned | `XDG_CACHE_HOME`, `XDG_DATA_HOME`, `CARGO_HOME`, `GOMODCACHE` and the like on every OS; protected folders move with them; roots recorded in the scan |
| [story-linux-catalog-flatpak-snap-and-the-tools-only-macos-lists](./stories/story-linux-catalog-flatpak-snap-and-the-tools-only-macos-lists.md) | planned | Research section, then rules for Flatpak caches and runtimes, Snap revisions, Steam, Rancher Desktop, Colima, Telegram, shader caches, Bazel, ccache and the like |
| [story-private-size-on-linux-reflinks-on-btrfs-and-xfs](./stories/story-private-size-on-linux-reflinks-on-btrfs-and-xfs.md) | planned | `private_bytes` from FIEMAP shared extents on reflink filesystems, within a budget; declares `private-size`; erratum to the research note |
| [story-whole-disk-on-linux-the-areas-outside-the-home](./stories/story-whole-disk-on-linux-the-areas-outside-the-home.md) | planned | Research section first; a Linux areas table with `needs-root` rows; root and rootless container engines measured; declares `whole-disk` |
| [story-freeing-space-outside-the-home-on-linux-ddc-disk-free](./stories/story-freeing-space-outside-the-home-on-linux-ddc-disk-free.md) | planned | docker and podman prune actions while the engine answers; root-only commands printed, never run |
| [story-homebrew-on-linux-ddc-disk-brew-packages-and-its-cleanup](./stories/story-homebrew-on-linux-ddc-disk-brew-packages-and-its-cleanup.md) | planned | Linuxbrew prefix, `cache.homebrew` at `~/.cache/Homebrew`, brew services on systemd; declares `homebrew` |
| [story-linux-test-parity-goldens-pinned-trees-distro-smoke-and-a-grouped-skip-list](./stories/story-linux-test-parity-goldens-pinned-trees-distro-smoke-and-a-grouped-skip-list.md) | planned | Linux goldens and load-tree hashes, the release archive smoke-tested on Debian, Fedora, Arch and Alpine, the skip list grouped by reason |

Suggested order:
1. The CI mounts job. The safety stories cannot close without it.
2. Safety, because these stories change what a clean may touch:
   1. mount boundaries and the Trash;
   2. btrfs and ZFS;
   3. file kinds;
   4. network and encrypted homes.
3. The product's main path on a developer's Linux machine:
   1. coding agents;
   2. the schedule, which decides the background environment;
   3. notifications.
4. The catalog:
   1. cache roots;
   2. the Linux catalog;
   3. private size.
5. Whole disk, then `disk free`, then Homebrew. Homebrew can move earlier, because it depends on neither.
6. Test parity, which closes the skip list once the capabilities have landed. Its goldens part can go earlier.

**Schema moves.** Several stories add a reason or a field to the JSON. Each story owns its own changes, and each moves the affected schema versions in its own PR, with the docs schemas and DDCKit:
- mount boundaries: the reasons `automount` and `trash-crosses-filesystem`;
- btrfs and ZFS: the reason `snapshot`;
- file kinds: the reasons `immutable` and `path-too-long`;
- network homes:
  - the id-mapping note on a root;
  - the `.nfs*` names in the clean report;
- cache roots: the roots used, in the scan report;
- private size: the Linux meaning of `private_bytes`;
- schedule: the reason `no-user-systemd`;
- whole disk:
  - the reason `needs-root`;
  - the new area ids;
  - the unreadable user-namespace count on a finding.

Two stories that land close together rebase. They never share one version move.

## Expected Results

- [ ] `CAPABILITIES` for Linux holds `in-use-check`, `private-size`, `whole-disk`, `coding-agents`, `homebrew`, `schedule` and `notifications`. The cases in `crates/ddc-cli/tests/cases.json` run these commands on Linux.
- [ ] No walk triggers an automount, enters a bind mount, or counts a snapshot. Each has a test on a real mount in CI, or an owner check with the reason.
- [ ] No clean copies a file into the Trash, opens a special file, or leaves a tombstone it cannot remove. That includes an immutable file, a foreign-owned file, a squashed NFS file and a path past `PATH_MAX`.
- [ ] A moved cache root is found, and the moved folder itself is never planned.
- [ ] `.config/native-skips/linux.txt` holds only its three groups, each with its reason.
- [ ] `ddc schedule install` on a desktop Linux session gives a timer that runs the background scan, and the scan shows a notification.

## Dependencies

- [[epics/DDC-007/epic|DDC-007 · Standalone CLI on macOS, Linux and Windows: portable core, release builds and installers]]: its platform seam, the Linux platform module and the Linux catalog. Its install story gives the Linux binary a stable path, which the schedule needs.
- [[epics/DDC-008/epic|DDC-008 · Native installers and the Windows tray app]]: story "Windows scheduled scans through Task Scheduler" plans the same scheduler seam in `cmd/schedule.rs`. Whichever lands first builds it.
- [[epics/DDC-003/epic|DDC-003]]: the coding agents' data and temporary folders, which story "Coding agents on Linux: ddc disk agents" ports.
- [[adr/cleanup-safety-model-risk-classes-plan-and-confirmation|Cleanup safety model: risk classes, plan and confirmation]]: §5 and §6 apply unchanged, except where the amendments named in the mount, btrfs and ZFS, file-kind, cache-root and `disk free` stories change them, each critic-approved.
- ADR "Platform seam: one platform module per OS with declared capabilities": each story that declares a capability updates the "when absent" column and the Linux row of §3.

## Open Questions

- [ ] A Linux desktop app or tray (GTK, Qt or a web view)? Out of this epic. The CLI with the timer and notifications is the Linux product until the owner decides otherwise.
- [ ] `.deb`, `.rpm`, AUR and Linuxbrew formulae are DDC-007's open question on package managers, not this epic's.
- [ ] Machines with several users: the in-use check sees only one's own processes (safety ADR, 2026-10-31 amendment). Does the whole-disk view show other users' homes as one row, or not at all? Story "Whole disk on Linux: the areas outside the home" decides.

## Related

- [[research/where-developer-disk-space-goes-linux|Where a developer's disk space goes (Linux)]]
- [[epics/DDC-007/stories/story-linux-scanner-and-rule-catalog|Linux scanner and rule catalog]]: the follow-ups it named are in this epic. They are the XDG cache-root token, root Docker's numbers, Flatpak and Snap, and Claude Code's `versions` folder.

---

*Last updated: 2026-11-01*
